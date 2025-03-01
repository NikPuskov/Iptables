ENV['VAGRANT_SERVER_URL'] = 'https://vagrant.elab.pro'

MACHINES = {
  :inetRouter => {
        :box_name => "ubuntu/jammy64",
        :vm_name => "inetRouter",
        :net => [   
                    ["192.168.255.1", 2, "255.255.255.252",  "router-net"], 
                ]
  },

  :inet2Router => {
       :box_name => "ubuntu/jammy64",
       :vm_name => "inet2Router",
       :net => [  
                    ["192.168.1.1",   2,  "255.255.255.252",  "server-net-2"],
                    ["192.168.56.2",  3,  "255.255.255.0",  ""],
	       ]

  },

  :centralRouter => {
        :box_name => "ubuntu/jammy64",
        :vm_name => "centralRouter",
        :net => [
                   ["192.168.255.2",  2, "255.255.255.252",  "router-net"],
                   ["192.168.0.1",    3, "255.255.255.252",  "server-net"],
                ]
  },

  :centralServer => {
        :box_name => "ubuntu/jammy64",
        :vm_name => "centralServer",
        :net => [
                   ["192.168.0.2",    2, "255.255.255.252",  "server-net"],
		   ["192.168.1.2",    3, "255.255.255.252",  "server-net-2"],
                ]
  }
}

Vagrant.configure("2") do |config|
  MACHINES.each do |boxname, boxconfig|
    config.vm.define boxname do |box|
      box.vm.box = boxconfig[:box_name]
      box.vm.host_name = boxconfig[:vm_name]
      
      box.vm.provider "virtualbox" do |v|
        v.memory = 768
        v.cpus = 1
       end

      boxconfig[:net].each do |ipconf|
	if ipconf[3] == ""
          box.vm.network("private_network", ip: ipconf[0], adapter: ipconf[1], netmask: ipconf[2])
	  next
        end
        box.vm.network("private_network", ip: ipconf[0], adapter: ipconf[1], netmask: ipconf[2], virtualbox__intnet: ipconf[3])
      end

      if boxconfig.key?(:public)
        box.vm.network "public_network", boxconfig[:public]
      end
     
      if boxconfig[:vm_name] == "centralServer"
       box.vm.provision "ansible" do |ansible|
        ansible.playbook = "ansible/provision.yaml"
	ansible.groups = {
  	  "routers" => ["inetRouter", "centralRouter", "inet2Router"]
	}
        ansible.limit = "all"
       end
      end
      

    end
  end
end
