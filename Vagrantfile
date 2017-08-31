# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
  required_plugins = %w(vagrant-timezone vagrant-hostmanager vagrant-proxyconf)
  plugins_to_install = required_plugins.select { |plugin| not Vagrant.has_plugin? plugin }
  if not plugins_to_install.empty?
    puts "Installing plugins: #{plugins_to_install.join(' ')}"
    if system "vagrant plugin install #{plugins_to_install.join(' ')}"
      exec "vagrant #{ARGV.join(' ')}"
    else
      abort "Installation of one or more plugins has failed. Aborting."
    end
  end

  config.vm.box = "bento/ubuntu-16.04"
  config.vm.box_check_update = false

  config.timezone.value = :host
  
  config.hostmanager.enabled = true
  config.hostmanager.manage_host = true
  config.hostmanager.manage_guest = true

  if ENV["http_proxy"]
    puts "http_proxy: " + ENV["http_proxy"]
    config.proxy.http = ENV["http_proxy"]
  end
  if ENV["https_proxy"]
    puts "https_proxy: " + ENV["https_proxy"]
    config.proxy.https = ENV["https_proxy"]
  end
  if ENV["no_proxy"]
    puts "no_proxy: " + ENV["no_proxy"]
    config.proxy.no_proxy = ENV["no_proxy"]
  end

  config.vm.define "jumpserver" do |node|
    node.vm.hostname = "jumpserver"
    #jumpserver.vm.network "forwarded_port", guest: 80, host: 8080
    node.vm.network "private_network", ip: "10.10.10.10"
    node.vm.provider "virtualbox" do |v|
      v.name = "jumpserver"
      v.cpus = "1"
      v.memory = "1024"
    end

    node.vm.provision "ansible_local" do |ansible|
      ansible.config_file = "ansible/ansible.cfg"
      ansible.playbook = "ansible/playbook.yml"
      ansible.inventory_path = "ansible/inventory.ini"
      ansible.sudo = true
      ansible.verbose = "v"
    end

    #jumpserver.vm.provision "file", source: "dissector_fuzz.sh", destination: "dissector_fuzz.sh"
    node.vm.provision "shell", path: "provision.sh"
  end
end