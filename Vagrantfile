# -*- mode: ruby -*-
# vi: set ft=ruby :

# All Vagrant configuration is done below. The "2" in Vagrant.configure
# configures the configuration version (we support older styles for
# backwards compatibility). Please don't change it unless you know what
# you're doing.
Vagrant.configure("2") do |config|
  # The most common configuration options are documented and commented below.
  # For a complete reference, please see the online documentation at
  # https://docs.vagrantup.com.

  # Every Vagrant development environment requires a box. You can search for
  # boxes at https://vagrantcloud.com/search.

  # config.vm.box = "generic/centos9s"
  # config.vm.box = "generic/alma9"
  config.vm.box = "generic/rocky9"

  config.vm.define "gitlab", primary: true do |cfg|
    # Disable automatic box update checking. If you disable this, then
    # boxes will only be checked for updates when the user runs
    # `vagrant box outdated`. This is not recommended.
    # config.vm.box_check_update = false

    # Create a forwarded port mapping which allows access to a specific port
    # within the machine from a port on the host machine. In the example below,
    # accessing "localhost:8080" will access port 80 on the guest machine.
    # NOTE: This will enable public access to the opened port
    # config.vm.network "forwarded_port", guest: 80, host: 8080

    # Create a forwarded port mapping which allows access to a specific port
    # within the machine from a port on the host machine and only allow access
    # via 127.0.0.1 to disable public access
    # config.vm.network "forwarded_port", guest: 80, host: 8080, host_ip: "127.0.0.1"
    cfg.vm.network "forwarded_port", guest: 22, host: 2222, host_ip: "127.0.0.1", id: "ssh", auto_correct: true
    # cfg.vm.network "forwarded_port", guest: 80, host: 8080, host_ip: "127.0.0.1", id: "http", auto_correct: true
    # cfg.vm.network "forwarded_port", guest: 443, host: 8443, host_ip: "127.0.0.1", id: "https", auto_correct: true

    # Create a private network, which allows host-only access to the machine
    # using a specific IP.
    # config.vm.network "private_network", ip: "192.168.33.10"

    # Create a public network, which generally matched to bridged network.
    # Bridged networks make the machine appear as another physical device on
    # your network.
    # config.vm.network "public_network"
    cfg.vm.network "private_network", ip: "192.168.56.82", id: :local

    # Share an additional folder to the guest VM. The first argument is
    # the path on the host to the actual folder. The second argument is
    # the path on the guest to mount the folder. And the optional third
    # argument is a set of non-required options.
    # config.vm.synced_folder "../data", "/vagrant_data"

    # Provider-specific configuration so you can fine-tune various
    # backing providers for Vagrant. These expose provider-specific options.
    # Example for VirtualBox:
    #
    cfg.vm.provider "virtualbox" do |vb|
      vb.name = "gitlab"
      # vb.linked_clone = true

      # # Display the VirtualBox GUI when booting the machine
      # vb.gui = true

      # Customize the amount of memory on the VM:
      vb.memory = 16 * 1024
      vb.cpus = 8
      vb.customize [ "modifyvm", :id, "--cpuexecutioncap", "85" ]
    end
    #
    # View the documentation for the provider you are using for more
    # information on available options.

    # Enable provisioning with a shell script. Additional provisioners such as
    # Ansible, Chef, Docker, Puppet and Salt are also available. Please see the
    # documentation for more information about their specific syntax and use.
    cfg.vm.provision "shell", name: "guest-yum-update", run: "once", privileged: true, inline: <<-SHELL
      echo "Updating the system packages (this step may take several minutes to complete)"
      date +"Start time: %F %T"
      (
        set -ex
        time yum -q -y update
      )
      date +"End time: %F %T"
    SHELL

    cfg.vm.provision "reload", name: "guest-reboot-with-upgraded-kernel", run: "once"

    cfg.vm.provision "host_shell", run: "once", inline: <<-SHELL
      sudo apt -qqq -y install jq
      default_route_json=$(ip -4 -j route | jq -c '.[] | select(.dst == "default")')
      device=$( jq -r '.dev'     <<< "${default_route_json}")
      address=$(jq -r '.prefsrc' <<< "${default_route_json}")
      rule_spec=(--in-interface "${device}" --destination "${address}" \
                  --protocol tcp --match multiport --destination-ports 80,443 \
                  --jump DNAT --to-destination 192.168.56.82)
      if ! sudo iptables --table nat --check PREROUTING "${rule_spec[@]}" > /dev/null; then
          sudo iptables --table nat --insert PREROUTING 1 "${rule_spec[@]}"
      fi
    SHELL

    cfg.vm.provision "shell", name: "guest-default-route", run: "always", privileged: true, inline: <<-SHELL
      echo "Switching default route from eth0 to eth1"
      set -ex
      route del default
      route add default gw 192.168.56.1
    SHELL
  end
end
