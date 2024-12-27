# -*- mode: ruby -*-
# vim: set ft=ruby :

MACHINES = {
  :node1 => {
        :box_name => "ubuntu/jammy64",
        :vm_name => "node1",
        :cpus => 2,
        :memory => 1024,
        :ip => "192.168.57.11",
        :wsl =>	2247,
  },
  :node2 => {
        :box_name => "ubuntu/jammy64",
        :vm_name => "node2",
        :cpus => 2,
        :memory => 1024,
        :ip => "192.168.57.12",
        :wsl =>	2248,

  },
  :barman => {
        :box_name => "ubuntu/jammy64",
        :vm_name => "barman",
        :cpus => 1,
        :memory => 1024,
        :ip => "192.168.57.13",
        :wsl =>	2249,

  },

}

Vagrant.configure("2") do |config|

  MACHINES.each do |boxname, boxconfig|

    config.vm.define boxname do |box|

      box.vm.box = boxconfig[:box_name]
      box.vm.host_name = boxconfig[:vm_name]
      box.vm.network "private_network", ip: boxconfig[:ip]
      box.vm.network "forwarded_port", auto_correct: true, guest: 22, host: boxconfig[:wsl], host_ip: "192.168.1.8", id: "ssh-for-wsl"
      box.vm.provider "virtualbox" do |v|
        v.memory = boxconfig[:memory]
        v.cpus = boxconfig[:cpus]
      end
    end
  end
end
