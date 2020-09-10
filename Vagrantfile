# vi: set ft=ruby :

ENV['VAGRANT_NO_PARALLEL'] = 'yes'

Vagrant.configure(2) do |config|
  
  config.vm.define "rabbitserver" do |rabbitserver|
    rabbitserver.vm.box = "centos/7"
    rabbitserver.vm.hostname = "rabbitserver.eduami.org"
    rabbitserver.vm.network "private_network", ip: "192.168.50.24"
    rabbitserver.vm.network "forwarded_port", guest: 5672, host: 5672
    rabbitserver.vm.network "forwarded_port", guest: 15672, host: 15672
    rabbitserver.vm.network "forwarded_port", guest: 9021, host: 9021
    rabbitserver.vm.network "forwarded_port", guest: 9092, host: 9092
    rabbitserver.vm.provision "shell", path: "startup-rabbitserver.sh"
    rabbitserver.vm.provider "virtualbox" do |vb|
      vb.name = "rabbitserver"
      vb.memory = 4024
      vb.cpus = 4
    end
  end
end
