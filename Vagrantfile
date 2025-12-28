Vagrant.configure("2") do |config|
  config.vm.provider "vmware_desktop" do |v|
    v.gui = true
    v.linked_clone = true
    # Force VMware à ne pas bloquer si le DHCP est lent
    v.vmx["ethernet0.addressType"] = "generated"
  end

  # 1. Admin (Ubuntu) - 192.168.56.10
  config.vm.define "admin" do |admin|
    admin.vm.box = "bento/ubuntu-22.04"
    admin.vm.network "private_network", ip: "192.168.56.10"
    admin.vm.hostname = "admin"
  end

  # 2 & 3. Noeuds RedHat - 192.168.56.11 & 12
  ["node01", "node02"].each_with_index do |name, i|
    config.vm.define name do |node|
      node.vm.box = "almalinux/9" # Clone RedHat gratuit et stable
      node.vm.network "private_network", ip: "192.168.56.1#{i+1}"
      node.vm.hostname = name
    end
  end

  # 4. Windows Server - 192.168.56.20
  config.vm.define "winsrv" do |win|
    win.vm.box = "StefanScherer/windows_2019"
    win.vm.network "private_network", ip: "192.168.56.20"
    win.vm.communicator = "winrm"
  end
end