# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|

  config.vm.box = "generic/centos7"
  config.vm.box_version = "4.3.8"
  config.vm.synced_folder ".", "/vagrant", type: "virtualbox"
  
  if Vagrant.has_plugin?("vagrant-timezone")
    config.timezone.value = :host
  end 

  config.vm.network "forwarded_port", guest: 9090, host: 9090
  config.vm.network "forwarded_port", guest: 9093, host: 9093
  config.vm.network "forwarded_port", guest: 9100, host: 9100
  config.vm.network "forwarded_port", guest: 3000, host: 3000
  config.vm.network "forwarded_port", guest: 80, host: 8080, host_ip: "127.0.0.1"
  config.vm.network "private_network", ip: "192.168.56.10"

  config.vm.provider "virtualbox" do |vb|
    vb.customize ["modifyvm", :id, "--natdnshostresolver1", "on"]
    vb.memory = "4096"
    vb.cpus = 2
    
  end
  config.vm.provision "shell", path: "script.sh" 
end
