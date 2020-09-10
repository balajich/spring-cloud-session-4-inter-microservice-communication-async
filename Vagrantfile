# vi: set ft=ruby :

ENV['VAGRANT_NO_PARALLEL'] = 'yes'

Vagrant.configure(2) do |config|
  
  config.vm.define "devserver" do |devserver|
    devserver.vm.box = "centos/7"
    devserver.vm.hostname = "devserver.eduami.org"
    devserver.vm.network "private_network", ip: "192.168.50.23"
    devserver.vm.network "forwarded_port", guest: 5672, host: 5672
    devserver.vm.network "forwarded_port", guest: 15672, host: 15672
    devserver.vm.provision "shell", path: "startup-devserver.sh"
    devserver.vm.provider "virtualbox" do |vb|
      vb.name = "devserver"
      vb.memory = 8024
      vb.cpus = 4
    end
  end
end
