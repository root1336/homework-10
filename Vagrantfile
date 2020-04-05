# -*- mode: ruby -*-
# vim: set ft=ruby :
home = ENV['HOME']

MACHINES = {
  :web2 => {
        :box_name => "centos/7",
        :ip_addr => '192.168.11.152',
  }
}

Vagrant.configure("2") do |config|

  MACHINES.each do |boxname, boxconfig|

      config.vm.define boxname do |box|

          box.vm.box = boxconfig[:box_name]
          box.vm.host_name = boxname.to_s

          box.vm.network "private_network", ip: boxconfig[:ip_addr]

          box.vm.provider :virtualbox do |vb|
          vb.customize ["modifyvm", :id, "--memory", "256"]
          vb.name = boxname.to_s

          end

	  box.vm.provision "shell", inline: <<-SHELL
	    mkdir -p ~root/.ssh
	    cp ~vagrant/.ssh/auth* ~root/.ssh
	  SHELL
          
          box.vm.provision "ansible" do |ansible|
            ansible.playbook = "playbooks/nginx-role.yml"
            ansible.become = "true"
          end
      end
   end
end
