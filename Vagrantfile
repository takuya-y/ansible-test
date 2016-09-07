# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure(2) do |config|
  config.vm.define "web" do |node|
        node.vm.box = "centos6.7"
        node.vm.hostname = "web"
        node.vm.network :private_network, ip: "192.168.100.20"
        node.vm.network :forwarded_port, id: "ssh", guest: 22, host: 2220
  end
  config.vm.define "database" do |node|
        node.vm.box = "centos6.7"
        node.vm.hostname = "database"
        node.vm.network :private_network, ip: "192.168.100.30"
        node.vm.network :forwarded_port, id: "ssh", guest: 22, host: 2230
  end
  config.vm.define "controller" do |node|
        node.vm.box = "centos6.7"
        node.vm.hostname = "controller"
        node.vm.network :private_network, ip: "192.168.100.10"
        node.vm.network :forwarded_port, id: "ssh", guest: 22, host: 2210
        node.vm.provision :shell, :inline => "yum install ansible -y"
        node.vm.provision :shell, :inline => "yum install git -y"
        node.vm.provision :shell, :inline => "ssh-keygen -t rsa -f /vagrant/.ssh/id_rsa -N ''"
  end
end
