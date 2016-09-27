Vagrant.configure(2) do |config|
  project_name = Dir.pwd.split('/').last
  config.vm.box = 'debian/contrib-jessie64'
  config.vm.box_check_update = false
  config.vm.provision :shell do |shell|
    ssh_pub_key = File.readlines("#{Dir.home}/.ssh/id_rsa.pub").first.strip
    shell.inline = <<-SHELL
      echo '' >> /home/vagrant/.ssh/authorized_keys
      echo #{ssh_pub_key} >> /home/vagrant/.ssh/authorized_keys
    SHELL
  end
  config.vm.provision :shell, path: 'bootstrap.sh'
  config.ssh.forward_agent = true

  config.vm.provider :virtualbox do |v|
    v.memory = 2048
  end

  config.hostmanager.enabled = true
  config.hostmanager.manage_host = true
  config.hostmanager.manage_guest = true
  config.hostmanager.ignore_private_ip = false

  config.vm.define :master, primary: true do |master|
    master.vm.synced_folder '.', '/vagrant', disabled: true
    master.vm.synced_folder '.', "/home/vagrant/#{project_name}"

    # Networking
    master.vm.network 'private_network', type: :dhcp
    master.vm.hostname = "node0.#{project_name}.local"
    master.hostmanager.aliases = ["node0.local"]
    master.hostmanager.ip_resolver = proc do |vm, resolving_vm|
      if hostname = (vm.ssh_info && vm.ssh_info[:host])
        `vagrant ssh -c "/sbin/ifconfig eth1" | grep "inet addr" | tail -n 1 | egrep -o "[0-9\.]+" | head -n 1 2>&1`.split("\n").first[/(\d+\.\d+\.\d+\.\d+)/, 1]
      end
    end

    # Port Forwarding
    master.vm.network 'forwarded_port', guest: 2376, host: 2376
    master.vm.network 'forwarded_port', guest: 2375, host: 2375
    master.vm.network 'forwarded_port', guest: 3000, host: 3000
    master.vm.network 'forwarded_port', guest: 4200, host: 4200
    master.vm.network 'forwarded_port', guest: 7357, host: 7357
    master.vm.network 'forwarded_port', guest: 35729, host: 35729

    # Configuration
    master.vm.provision :ansible_local do |ansible|
      ansible.install = false
      ansible.playbook = 'dev.yml'
      ansible.provisioning_path = "/home/vagrant/#{project_name}/ansible"
      ansible.inventory_path = 'inventory'
    end
  end

  (1..3).each do |i|
    config.vm.define "node#{i}", autostart: false do |node|
      node.vm.synced_folder '.', '/vagrant', disabled: true
      node.vm.synced_folder '.', "/home/vagrant/#{project_name}"

      # Networking
      node.vm.hostname = "node#{i}.#{project_name}.local"
      node.vm.network 'private_network', type: :dhcp
      node.hostmanager.aliases = ['node1.local']
      node.hostmanager.ip_resolver = proc do |vm, resolving_vm|
        if hostname = (vm.ssh_info && vm.ssh_info[:host])
          `vagrant ssh node#{i} -c "/sbin/ifconfig eth1" | grep "inet addr" | tail -n 1 | egrep -o "[0-9\.]+" | head -n 1 2>&1`.split("\n").first[/(\d+\.\d+\.\d+\.\d+)/, 1]
        end
      end

      # Configuration
      node.vm.provision :ansible_local do |ansible|
        ansible.install = false
        ansible.playbook = 'cluster.yml'
        ansible.provisioning_path = "/home/vagrant/#{project_name}/ansible"
        ansible.inventory_path = 'inventory'
        ansible.limit = "node#{i}.local"
      end
    end
  end

  if Vagrant.has_plugin?('vagrant-cachier')
    config.cache.scope = :box
  end
end
