# -*- mode: ruby -*-
# vi: set ft=ruby :

FILTER_NAME='YubiKey 4'
MANUFACTURER='Yubico'
VENDOR_ID='0x1050'
PRODUCT_ID='0x0406'
PRODUCT='YubiKey FIDO+CCID'

Vagrant.configure('2') do |config|
  config.vagrant.plugins = 'vagrant-hostmanager'
  config.hostmanager.enabled = true
  config.hostmanager.manage_host = false
  config.hostmanager.manage_guest = true
  config.hostmanager.ignore_private_ip = false
  config.hostmanager.include_offline = true

  config.vm.box = 'ubuntu/focal64'

  # The target machine is used for testing that awx is able
  # to apply a playbook against a host.
  config.vm.define 'target' do |target| 
    target.vm.network 'private_network', ip:'192.168.56.20'
    target.vm.hostname = 'target'
    target.vm.provider :virtualbox do |vb|
       vb.name = 'target'
       vb.memory = 512
       vb.cpus = 1
    end
  end

  config.vm.define 'awx' do |awx| 
    awx.vm.network 'private_network', ip:'192.168.56.10'
    awx.vm.hostname = 'awx'
    awx.vm.provider :virtualbox do |vb|
       vb.name = 'awx'
    end
  
    awx.vm.provider 'virtualbox' do |vb|
      vb.memory = 6144
      vb.cpus = 4
  
      vb.customize ['modifyvm', :id, '--usb', 'on']
      vb.customize ['modifyvm', :id, '--usbehci', 'on']
      vb.customize [
        'usbfilter', 'add', '0',
        '--target', :id,
        '--name', 'YubiKey 4',
        '--manufacturer', 'Yubico',
        '--vendorid', '0x1050',
        '--productid', '0x0406',
        '--product', 'YubiKey FIDO+CCID'
      ]
  
    end

    awx.vm.provision :ansible do |ansible|
      ansible.playbook = 'ansible/vagrant.yml'
      ansible.galaxy_role_file = 'ansible/requirements.yml'
      ansible.galaxy_roles_path = 'ansible/ansible-galaxy-roles'
      ansible.galaxy_command = 'ansible-galaxy install --role-file=%{role_file} --roles-path=%{roles_path}'
      ansible.extra_vars = {
        'awx_host' => '192.168.56.10',
      }
      ansible.limit = 'all'
      ansible.groups = {
        'worker' => ['worker'],
        'awx' => ['awx'],
      }
    end
  end
end
