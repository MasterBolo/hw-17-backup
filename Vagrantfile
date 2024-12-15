# -*- mode: ruby -*-
# vi: set ft=ruby :

disk = './secondDisk.vdi'

Vagrant.configure(2) do |config|
if Vagrant.has_plugin?("vagrant-vbguest")
      config.vbguest.auto_update = false
      config.vbguest.no_remote = true
 end  
 config.vm.box = "bento/ubuntu-22.04"  
 config.vm.provider "virtualbox" do |v| 
 v.memory = 1024
 v.cpus = 1
 end 
 config.vm.define "backupsrv" do |backupsrv| 
 backupsrv.vm.network "private_network", ip: "192.168.56.10",  virtualbox__intnet: "net1" 
 backupsrv.vm.hostname = "backupsrv"
 backupsrv.vm.provider "virtualbox" do |vb|
   unless File.exist?(disk)
    vb.customize ['createhd', '--filename', disk, '--variant', 'Fixed', '--size', 2 * 1024]
 end
    vb.customize ['storageattach', :id,  '--storagectl', 'SATA Controller', '--port', 2, '--device', 0, '--type', 'hdd', '--medium', disk]
 end

 backupsrv.vm.provision "ansible" do |ansible|
    ansible.playbook = "Ansible/backupsrv.yml" 
   end
  end

 config.vm.define "client" do |client| 
 client.vm.network "private_network", ip: "192.168.56.11",  virtualbox__intnet: "net1" 
 client.vm.hostname = "client"
 client.vm.provision "ansible" do |ansible|
    ansible.playbook = "Ansible/client.yml"
    end
  end
end



  


