# k8s-ha-lab: a highly available Kubernetes cluster, from scratch

A production-style Kubernetes cluster built with **kubeadm** on 5 KVM virtual machines:
3 control planes with stacked etcd behind a floating **kube-vip** IP, 2 workers, and
**Cilium** (eBPF) replacing kube-proxy. The whole lab is code: `vagrant up` plus three
Ansible playbooks rebuild it from nothing.

The goal is to practice what you only learn on a real HA cluster: losing a master,
etcd quorum, backup/restore and upgrades without downtime.

## Architecture

```
                 VIP 192.168.56.100:6443  (kube-vip, ARP, leader election)
                 /          |           \
          cp1 (.11)    cp2 (.12)    cp3 (.13)     control plane + stacked etcd
                 \          |           /
                w1 (.21)          w2 (.22)        workers
```

| Component | Choice | Why |
|---|---|---|
| Hypervisor | KVM/libvirt + Vagrant | Native on Linux, lighter than VirtualBox, VMs as code |
| OS | Ubuntu 24.04 (`alvistack/ubuntu-24.04` box) | LTS, maintained libvirt box |
| Kubernetes | v1.37 (installed as v1.36, then upgraded in place) | The upgrade is part of the lab |
| Runtime | containerd, systemd cgroup driver | kubeadm default, matches kubelet |
| API endpoint | kube-vip static pods, ARP mode | No extra load balancer VM; the VIP follows a healthy master |
| etcd | Stacked, 3 members | Tolerates the loss of 1 member (quorum = 2) |
| CNI | Cilium 1.20, `kubeProxyReplacement=true` | Services in eBPF, no kube-proxy/iptables; Hubble for flow visibility |
| Automation | Ansible, idempotent playbooks | Re-runnable, the cluster can be rebuilt from zero |

Resources: 3 x (2 vCPU, 2 GB) + 2 x (2 vCPU, 1.5 GB), about 9 GB of RAM on a 16 GB laptop.

## Quick start

Requirements on the host: KVM/libvirt, Vagrant with the `vagrant-libvirt` plugin, Ansible.

```bash
vagrant up                                  # 5 VMs
cd ansible
ansible-playbook prepare-nodes.yml          # swap, kernel, containerd, kubeadm/kubelet/kubectl
ansible-playbook bootstrap-cluster.yml      # kube-vip + kubeadm init, join masters and workers
ansible-playbook install-cilium.yml         # CNI, kube-proxy replacement

export KUBECONFIG=$PWD/../kubeconfig        # admin kubeconfig, points at the VIP (git-ignored)
kubectl get nodes
```

Tear down: `vagrant destroy -f`.

## Repository layout

```
Vagrantfile                      5 VMs, static IPs on a private network
ansible/
  inventory.ini                  nodes, VIP, versions, subnets
  prepare-nodes.yml              node prerequisites (phase 2)
  bootstrap-cluster.yml          kube-vip, kubeadm init, joins (phases 3, 4, 6)
  install-cilium.yml             Cilium (phase 5)
  etcd-restore.yml               disaster recovery: restore etcd on the 3 masters
  upgrade-cluster.yml            rolling minor upgrade, node by node
  templates/kubeadm-init.yaml.j2 kubeadm InitConfiguration + ClusterConfiguration
  tasks/user-kubeconfig.yml
```

## HA test: losing the master that holds the VIP

Method: an `nginx` Deployment (3 replicas, NodePort) runs on the workers. A loop queries
`/readyz` through the VIP every second from the host. The VIP holder (`cp1`) is then
powered off abruptly with `virsh destroy` (equivalent to pulling the plug).

| Measure | Result |
|---|---|
| VIP holder before / after | `cp1` → `cp2` |
| API unavailable through the VIP | about **20 s** (15:43:47 → 15:44:07) |
| Application traffic (NodePort) | not interrupted |
| etcd after the loss | 2/3 members healthy, quorum kept, new leader elected (`cp3`) |

The ~20 s are the time for the kube-vip lease to expire, for a surviving master to win
the election and announce the VIP with a gratuitous ARP, and for etcd to elect a new leader.
The test loop has a 2 s timeout plus a 1 s pause, so the measure is accurate to ~3 s.

## etcd drills

**Snapshot.** `etcdctl snapshot save` on a master, integrity checked with
`etcdutl snapshot status`, then copied off the cluster to `backups/` (git-ignored: an etcd
snapshot holds every Secret in plain text).

**Replacing a dead master.** `cp3` was destroyed, then:
1. quorum checked from `cp1`: 2/3 members healthy, the cluster keeps working;
2. dead member removed with `etcdctl member remove` (otherwise a new `cp3` can't join:
   etcd refuses a member whose name/peer URL already exists);
3. `kubectl delete node cp3`, then the VM was rebuilt from scratch with `vagrant up` +
   `prepare-nodes.yml` + `bootstrap-cluster.yml`;
4. `cp3` back in the quorum with a **new member ID** (`31197a5d…` → `af873236…`).

**Full restore** (`ansible/etcd-restore.yml`). Unlike a single-node restore, a 3-member
stacked etcd needs every member restored together:
1. stop all etcd and kube-apiserver static pods on the 3 masters;
2. move each data dir aside (kept, never deleted during a DR);
3. `etcdutl snapshot restore` on each master with its own `--name` and peer URL and a
   shared `--initial-cluster`: this creates a new 3-member cluster from the snapshot data;
4. start everything again, restart controller-manager, scheduler and kubelets.

Test: a namespace present in the snapshot was deleted, and another one created after the
snapshot. After the restore, the first one is back with its Deployment, and the second one
is gone. Anything written after the last snapshot is lost: the backup frequency is the RPO.

Lesson learned: the first run of the restore runbook stopped halfway, with the API down on
all masters, because it relied on `crictl`, which isn't installed on the nodes. It now
watches processes with `pgrep`, and it is idempotent, so re-running it resumed the restore
where it stopped. Test DR runbooks before you need them.

## Rolling upgrade: 1.36 → 1.37

`ansible/upgrade-cluster.yml` follows the kubeadm procedure, one node at a time
(`serial: 1`): switch the apt repo to the new minor, upgrade kubeadm, then
`kubeadm upgrade apply` on the first master (control plane, etcd, CoreDNS) or
`kubeadm upgrade node` on the others, then drain, upgrade kubelet and kubectl, wait for
`Ready`, uncordon. Nodes already on the target version are skipped, so it can be re-run.

Before starting: a fresh etcd snapshot, and a PodDisruptionBudget on the test app
(`minAvailable: 2` out of 3 replicas), so a drain evicts its pods one at a time and waits
for each replacement to be Ready.

Kubeadm keeps kube-proxy out of the upgrade: skipping it at init is recorded as
`proxy.disabled` in the cluster configuration, and the upgrade's kube-proxy phase checks it.

| Measure (polled every second during the whole upgrade) | Result |
|---|---|
| Application (NodePort, 3 replicas, PDB) | **always HTTP 200**, no interruption |
| API through the VIP | one window of about **22 s** unavailable (00:29:06 → 00:29:29) |
| End state | 5 nodes on v1.37, all Ready |

The API window happened while the API server restarted on the master that held the VIP.
In ARP mode, kube-vip does not health-check the API server: the VIP only moves once the
leader fails to renew its lease, which takes about as long as the failover test. Pods
keep running without the API, so applications are not affected. To remove that window,
the options are a load balancer with real health checks (HAProxy on `/readyz`, BGP/ECMP),
or moving the VIP away from a master before upgrading it.

## Problems met and how they were solved

1. **`generic/ubuntu2404` box returns 404.** The Roboxes `generic/*` boxes are no longer
   available, and Canonical stopped publishing official Vagrant boxes with 24.04.
   Switched to `alvistack/ubuntu-24.04`, maintained and built for libvirt.

2. **Two network interfaces per VM.** Vagrant adds a NAT interface (default route) next to
   the private network. Without care, components pick the NAT IP:
   - kubelet would register the NAT IP as `InternalIP` → fixed with `--node-ip` in
     `/etc/default/kubelet`;
   - kubeadm advertises the default-route IP → `advertiseAddress` set explicitly in the
     kubeadm config and `--apiserver-advertise-address` on joins.

3. **kube-vip cannot start during `kubeadm init` (Kubernetes ≥ 1.29).** Since 1.29,
   `admin.conf` only gets its rights once init has finished, so kube-vip cannot take its
   leader-election lease and the VIP never comes up. Fix: the first master's kube-vip
   manifest uses `super-admin.conf` during init, then is switched back to `admin.conf`
   (`super-admin.conf` exists only on the first master). See kube-vip issue #684.

4. **`kubeadm init phase upload-certs` times out after exactly 60 s.** Reading the kubeadm
   source showed that with the default `/etc/kubernetes/admin.conf`, kubeadm rewrites the
   API address to the node's local endpoint, which defaults to the default-route IP (the
   NAT IP). That IP is not in the API server certificate, so every call fails TLS
   verification and is retried until timeout. Fix: pass `--kubeconfig` with another path,
   which kubeadm uses as-is (through the VIP, which is in the certificate).

5. **Delegated tasks connecting with the wrong SSH key.** The drain runs on the first
   master on behalf of the node being upgraded (`delegate_to`). The key path was a group
   variable built from `{{ inventory_hostname }}`, which still refers to the original host
   during a delegated task, so Ansible connected to `cp1` with `cp2`'s key. Fix: an
   explicit key path on each host line of the inventory.

## Roadmap

- [x] Node preparation automated with Ansible
- [x] HA control plane: 3 masters, stacked etcd, kube-vip VIP
- [x] Cilium with kube-proxy replacement
- [x] HA test: loss of the VIP holder
- [x] etcd: snapshot, restore, replacement of a quorum member
- [x] Minor version upgrade (1.36 → 1.37), node by node, without workload downtime
- [ ] NetworkPolicies: default-deny per namespace, explicit allows, observed with Hubble
