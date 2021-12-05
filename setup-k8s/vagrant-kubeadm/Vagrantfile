# -*- mode: ruby -*-
# vi:set ft=ruby sw=2 ts=2 sts=2:

NUM_MASTER_NODE = 1
NUM_WORKER_NODE = 2

IP_NW = "192.168.56."
MASTER_IP_START = 1
NODE_IP_START = 2
Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/bionic64"
  # Disable automatic box update checking.
  config.vm.box_check_update = false
  config.vm.provision "shell", path: "ubuntu/initialSetup.sh"

  # Provision Master Nodes
  (1..NUM_MASTER_NODE).each do |i|
      config.vm.define "master0#{i}" do |node|
        node.vm.provider "virtualbox" do |vb|
            vb.name = "master0#{i}"
            vb.memory = 2048
            vb.cpus = 2
        end
        node.vm.hostname = "master0#{i}"
        node.vm.network :private_network, ip: IP_NW + "#{MASTER_IP_START + i}"
        node.vm.network "forwarded_port", guest: 22, host: "#{2710 + i}"
        node.vm.provision "setup-hosts", :type => "shell", :path => "ubuntu/setup-hosts.sh" do |s|
          s.args = ["enp0s8"]
        end
        node.vm.provision "setup-dns", type: "shell", :path => "ubuntu/update-dns.sh"
        node.vm.provision "shell", type: "shell", path: "ubuntu/setupmaster.sh"
      end
  end

  # Provision Worker Nodes
  (1..NUM_WORKER_NODE).each do |i|
    config.vm.define "worker0#{i}" do |node|
        node.vm.provider "virtualbox" do |vb|
            vb.name = "worker0#{i}"
            vb.memory = 1024
            vb.cpus = 2
        end
        node.vm.hostname = "worker0#{i}"
        node.vm.network :private_network, ip: IP_NW + "#{NODE_IP_START + i}"
        node.vm.network "forwarded_port", guest: 22, host: "#{2720 + i}"
        node.vm.provision "setup-hosts", :type => "shell", :path => "ubuntu/setup-hosts.sh" do |s|
          s.args = ["enp0s8"]
        end
        node.vm.provision "setup-dns", type: "shell", :path => "ubuntu/update-dns.sh"
        node.vm.provision "setup-dns", type: "shell", :path => "ubuntu/setupworker.sh"
    end
  end
end
