# -*- mode: ruby -*-
# vi: set ft=ruby :
VAGRANTFILE_API_VERSION = '2'
Vagrant.require_version '>= 1.8.3'

require 'yaml'

# Paths.
vagrant_dir = File.dirname(File.expand_path(__FILE__))

Vagrant.configure(VAGRANTFILE_API_VERSION) do |v|
  # Load default VM configuration.
  config = YAML.load_file("#{vagrant_dir}/servers/default.config.yml")

  # Load VM server configurations.
  Dir.entries("#{vagrant_dir}/servers").select {|entry| File.directory? File.join("#{vagrant_dir}/servers", entry) and !(entry =='.' || entry == '..') }.each do |dir|
    server_dir = "#{vagrant_dir}/servers/#{dir}"

    # Merge server configuration.
    config.merge!(YAML.load_file("#{server_dir}/config.yml"))

    # Merge site configurations.
    Dir.entries("#{server_dir}/sites").select {|entry| !(entry == '.' || entry == '..') }.each do |file|
      site = YAML::load_file("#{server_dir}/sites/#{file}")

      # Explicitly merge synced folders, sites, vhosts, mysql databases and
      # users. Other keys in site configurations are ignored.
      ['vagrant_synced_folders',
       'sites',
       'apache_vhosts',
       'mysql_databases',
       'mysql_users'].each do |k|
        site[k].each do |value|
          config[k] << value
        end
      end
    end

    # Configure servers.
    v.vm.define config['name'] do |s|
      # SSH config.
      s.ssh.insert_key = false
      s.ssh.forward_agent = true

      # VM config.
      s.vm.box = config['vagrant_box']
      s.vm.hostname = config['vagrant_hostname']
      s.vm.network :private_network, ip: config['vagrant_ip']

      # Synced folders.
      config['vagrant_synced_folders'].each do |synced_folder|
        options = {
          type: synced_folder['type'],
          rsync__auto: 'true',
          rsync__exclude: synced_folder['excluded_paths'],
          rsync__args: ['--verbose', '--archive', '--delete', '-z', '--chmod=ugo=rwX'],
          id: synced_folder['id'],
          create: synced_folder.include?('create') ? synced_folder['create'] : false,
          mount_options: synced_folder.include?('mount_options') ? synced_folder['mount_options'] : []
        }
        if synced_folder.include?('options_override')
          options = options.merge(synced_folder['options_override'])
        end
        s.vm.synced_folder synced_folder['local_path'], synced_folder['destination'], options
      end

      # Virtualbox config.
      s.vm.provider :virtualbox do |p|
        p.name = config['name']
        p.memory = config['vagrant_memory']
        p.customize ['modifyvm', :id, '--ioapic', 'on']

        # See https://forums.virtualbox.org/viewtopic.php?f=7&t=50368
        p.customize ['modifyvm', :id, '--natdnshostresolver1', 'on']
      end

      # Ansible provisioning.
      s.vm.provision 'ansible' do |a|
        a.playbook = "#{vagrant_dir}/ansible/playbook.yml"
        a.galaxy_role_file = "#{vagrant_dir}/ansible/requirements.yml"
        a.extra_vars = config
      end
    end
  end
end
