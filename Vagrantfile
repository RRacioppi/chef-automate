VAGRANT_BOX = 'bento/ubuntu-16.04'
BOX_RAM_MB_AUTOMATE = '4096'
BOX_RAM_MB_SERVER = '4096'
BOX_RAM_MB_NODE = '1024'
BOX_CPUS = '2'
CPU_CAP = '70'
CHEF_AUTOMATE_IP = '192.168.2.199'
CHEF_SERVER_IP = '192.168.2.200'
CHEF_NODE_IP = '192.168.2.201'
CHEF_AUTOMATE_HOSTNAME = 'chef-automate.test'
CHEF_SERVER_HOSTNAME = 'chef-server.test'
CHEF_NODE_HOSTNAME = 'chef-node'
CHEF_AUTOMATE_ZIP_URL = 'https://packages.chef.io/files/current/automate/latest/chef-automate_linux_amd64.zip'
CHEF_SERVER_ZIP_URL = 'https://packages.chef.io/files/stable/chef-server/12.19.31/ubuntu/18.04/chef-server-core_12.19.31-1_amd64.deb'

Vagrant.configure("2") do |rootconfig|
  rootconfig.vm.define "chef-node" do |chef_node|
    default_provision(chef_node, CHEF_NODE_HOSTNAME, CHEF_NODE_IP, BOX_CPUS, BOX_RAM_MB_NODE, CPU_CAP)
    chef_node.vm.provision "shell", inline: 'set -x; apt-get update'
  end

  rootconfig.vm.define "chef-automate" do |chef_automate|
    default_provision(chef_automate, CHEF_AUTOMATE_HOSTNAME, CHEF_AUTOMATE_IP, BOX_CPUS, BOX_RAM_MB_AUTOMATE, CPU_CAP)
    chef_automate.vm.provision "shell", inline: <<-SHELL
      set -x;
      apt-get update && apt-get install -y unzip
      sysctl -w vm.max_map_count=262144
      sysctl -w vm.dirty_expire_centisecs=20000
      # INSTALLAZIONE AUTOMATE
      curl #{CHEF_AUTOMATE_ZIP_URL} | gunzip - > chef-automate && chmod +x chef-automate
      yes y | sudo ./chef-automate deploy
    SHELL
  end

  rootconfig.vm.define 'chef-server' do |chef_server|       
    default_provision(chef_server, CHEF_SERVER_HOSTNAME, CHEF_SERVER_IP, BOX_CPUS, BOX_RAM_MB_SERVER, CPU_CAP)
    chef_server.vm.provision "shell", inline: <<-SHELL
    set -x;     
      apt-get update && sudo apt-get install -y sshpass
      # INSTALLAZIONE CHEF
      curl #{CHEF_SERVER_ZIP_URL} --output chef-server-core.deb
      sudo dpkg -i chef-server-core.deb
      sudo chef-server-ctl reconfigure
      sudo chef-server-ctl user-create admin admin admin admin@admin.com password --filename /etc/chef/admin.pem
      sudo chef-server-ctl org-create default_organization "Default Organization" --association_user admin --filename /etc/chef/validator.pem
      sudo chef-server-ctl install chef-manage
      sudo chef-server-ctl reconfigure
      sudo chef-manage-ctl reconfigure --accept-license
      # INTEGRAZIONE AUTOMATE
      yes | openssl s_client -connect #{CHEF_AUTOMATE_HOSTNAME}:443 | sudo openssl x509 -out /usr/local/share/ca-certificates/chef-automate.crt
      sudo update-ca-certificates
        mkdir -p ~/.ssh ; ssh-keyscan -H #{CHEF_AUTOMATE_IP} >> ~/.ssh/known_hosts
      ADMIN_TOKEN=$(sshpass -p vagrant ssh vagrant@#{CHEF_AUTOMATE_IP} "sudo chef-automate admin-token")
      sudo chef-server-ctl set-secret data_collector token ${ADMIN_TOKEN}
      sudo chef-server-ctl restart nginx
      sudo chef-server-ctl restart opscode-erchef
      echo "data_collector['root_url'] = 'https://#{CHEF_AUTOMATE_HOSTNAME}/data-collector/v0/'" | sudo tee -a /etc/opscode/chef-server.rb
      echo "data_collector['proxy'] = true" | sudo tee -a /etc/opscode/chef-server.rb
      echo "profiles['root_url'] = 'https://#{CHEF_AUTOMATE_HOSTNAME}'" | sudo tee -a /etc/opscode/chef-server.rb
      sudo chef-server-ctl reconfigure
      # BOOTSTRAP DEL NODO CHEF (INSTALLAZIONE DI KNIFE)
      curl -L https://omnitruck.chef.io/install.sh | sudo bash
      mkdir -p ~/.chef
      echo "chef_server_url 'https://#{CHEF_SERVER_HOSTNAME}/organizations/default_organization'" | tee -a ~/.chef/config.rb
      echo "client_key '/etc/chef/admin.pem'" |  tee -a ~/.chef/config.rb
      echo "node_name 'admin'" |  tee -a ~/.chef/config.rb
      echo "ssl_verify_mode :verify_none" |  tee -a ~/.chef/config.rb
      knife ssl fetch
      yes y | knife bootstrap #{CHEF_NODE_IP} -N #{CHEF_NODE_HOSTNAME} --connection-user vagrant -P vagrant --sudo --verbose --no-host-key-verify --chef-license accept
    SHELL
  end
end

def default_provision(virtual_machine, hostname, ip, cpu, ram, cpu_cap)
  virtual_machine.vm.box = VAGRANT_BOX
  virtual_machine.vm.hostname = hostname
  virtual_machine.vm.network 'private_network', ip: ip
  virtual_machine.vm.provider :virtualbox do |p|
    p.name = hostname
    p.linked_clone = true
    p.customize ["modifyvm", :id, "--cpus", cpu]
    p.customize ["modifyvm", :id, "--memory", ram]
    p.customize ["modifyvm", :id, "--cpuexecutioncap", cpu_cap]
  end
  virtual_machine.berkshelf.enabled = false if Vagrant.has_plugin?("vagrant-berkshelf")
  virtual_machine.vm.provision "shell", inline: <<-SHELL
    set -x;
    echo #{CHEF_AUTOMATE_IP} #{CHEF_AUTOMATE_HOSTNAME} | sudo tee -a /etc/hosts
    echo #{CHEF_SERVER_IP} #{CHEF_SERVER_HOSTNAME} | sudo tee -a /etc/hosts
    echo #{CHEF_NODE_IP} #{CHEF_NODE_HOSTNAME} | sudo tee -a /etc/hosts
  SHELL
end