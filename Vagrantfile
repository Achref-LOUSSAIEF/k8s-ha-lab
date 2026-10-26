# -*- mode: ruby -*-
# HA Kubernetes lab: 3 control planes + workers, on KVM/libvirt.
# Vagrant only creates the VMs; Ansible prepares them (see ansible/).

BOX     = "generic/ubuntu2404"
NETWORK = "192.168.56"   # private network; the API VIP will be 192.168.56.100

# Comment out w2 if you are short on RAM (remove it from ansible/inventory.ini too).
NODES = [
  { name: "cp1", ip: "#{NETWORK}.11", cpus: 2, mem: 2048 },
  { name: "cp2", ip: "#{NETWORK}.12", cpus: 2, mem: 2048 },
  { name: "cp3", ip: "#{NETWORK}.13", cpus: 2, mem: 2048 },
  { name: "w1",  ip: "#{NETWORK}.21", cpus: 2, mem: 1536 },
  { name: "w2",  ip: "#{NETWORK}.22", cpus: 2, mem: 1536 },
]

Vagrant.configure("2") do |config|
  config.vm.box = BOX
  config.vm.synced_folder ".", "/vagrant", disabled: true

  NODES.each do |n|
    config.vm.define n[:name] do |node|
      node.vm.hostname = n[:name]
      node.vm.network :private_network, ip: n[:ip]

      node.vm.provider :libvirt do |lv|
        lv.cpus   = n[:cpus]
        lv.memory = n[:mem]
      end
    end
  end
end
