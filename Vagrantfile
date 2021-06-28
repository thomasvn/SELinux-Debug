# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
  config.vm.box = "centos/7"

  config.vm.provision "shell", inline: <<-SHELL
    sudo yum -y install podman vim
    echo "alias sp='sudo podman'" >> ~/.bashrc
  SHELL
end
