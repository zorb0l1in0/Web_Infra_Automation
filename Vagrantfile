Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/jammy64"
  config.vm.hostname = "poc-docker"

  # HAProxy listening on port 8080 on the host machine and forwarding to port 80 on the guest machine
  config.vm.network "forwarded_port", guest: 80, host: 8080

  config.vm.provider "virtualbox" do |vb|
    vb.memory = 2048
    vb.cpus = 2
  end
  
  # Provisioning with Ansible
  config.vm.provision "ansible_local" do |ansible|
    ansible.playbook = "playbook.yml"
    ansible.install_mode = :pip_args_only
    ansible.pip_args = "ansible-core==2.17.*"
    ansible.galaxy_role_file = "requirements.yml"
    ansible.galaxy_command = "ansible-galaxy install -r %{role_file}"
    ansible.compatibility_mode = "2.0"
  end
end