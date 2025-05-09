Vagrant.configure("2") do |config|

  # Based on Kubernetes The Hard Way
  boxes = [
    { :name => "jumpbox", :cpus => 1, :memory => 512,  :disk => "10GB" },
    { :name => "server",  :cpus => 1, :memory => 2048, :disk => "20GB" },
    { :name => "node-0",  :cpus => 1, :memory => 2048, :disk => "20GB",
      :podsubnet => "10.200.0.0/24" },
    #{ :name => "node-1",  :cpus => 1, :memory => 2048, :disk => "20GB",
    #  :podsubnet => "10.200.1.0/24" },
  ]


  # Create a 'routes' file which contains a mapping of pod subnets to the node
  # hostname. At runtime, we'll create static routes.
  config.trigger.before :up do |trigger|
    trigger.name = "Output routes file"
    trigger.ruby do |env,machine|
      Dir.mkdir("shared") unless File.exist?("shared")
      File.open("shared/routes", "w") do |f|
        boxes.each do |b|
          f.puts "#{b[:podsubnet]} #{b[:name]}" if b[:podsubnet]
        end
      end
    end
  end


  boxes.each do |opts|
    config.vm.define opts[:name], primary: opts[:name] == "jumpbox" do |node|
      node.vm.box = "bento/ubuntu-24.04"
      node.vm.box_version = "202502.21.0"
      node.vm.hostname = opts[:name]

      # Just explicitly noting here that this directory is, by default, exposed
      # as a synced folder at /vagrant.
      node.vm.synced_folder '.', '/vagrant', disabled: false

      # We can't shrink the disk smaller than the original, so just skip this.
      # Also note we would have to set linked_clone = false if we want to resize.
      false and node.vm.disk :disk, size: opts[:disk], primary: true

      node.vm.provider "vmware_desktop" do |vmw|
        vmw.vmx["memsize"] = opts[:memory]
        vmw.vmx["numvcpus"] = opts[:cpus]
      end

      # Quiet the motd
      node.vm.provision "shell", inline: <<-SHELL
        set -eux
        cd /etc/update-motd.d/
        rm -f 10-help-text 50-motd-news 91-contract-ua-esm-status 99-bento
      SHELL


      # -- Networking ----------------------------------------------------------

      # Configure a private network shared by all the nodes.
      #
      # As of Big Sur, we cannot specify the IP address to take.
      # https://developer.hashicorp.com/vagrant/docs/providers/vmware/known-issues#big-sur-related-issues
      node.vm.network :private_network

      # Assign a FQDN to each of the nodes.
      #
      # XXX: I'm not sure why the VMs have a hosts entry `127.0.1.1 vagrant`
      # and use 127.0.2.1 for the actual hostname. This deviates from the setup
      # in Kubernetes The Hard Way.
      opts[:name] != "jumpbox" and
      node.vm.provision "shell", inline: <<-SHELL
        set -eux
        HOSTNAME=$(hostname --short)
        FQDN=${HOSTNAME}.kubernetes.local
        sed -i "s/^127.0.2.1.*/127.0.2.1 ${FQDN} ${HOSTNAME}/" /etc/hosts
        test "$(hostname --fqdn)" = "$FQDN"
      SHELL

      # Generate an SSH key on the jumpbox for the vagrant user
      opts[:name] == "jumpbox" and
      node.vm.provision "shell", privileged: false, inline: <<-SHELL
        set -eux
        ssh-keygen -t rsa -f ~/.ssh/id_rsa -N ""
        cp ~/.ssh/id_rsa.pub /vagrant/shared/jumpbox_ssh_key.pub
      SHELL

      # All other boxes: Install service for copying the key to authorized_keys
      opts[:name] != "jumpbox" and node.vm.provision "shell", inline: <<-SHELL
        set -eux
        cp /vagrant/files/etc/systemd/system/install-jumpbox-ssh-key.service \
          /etc/systemd/system/
        systemctl daemon-reload
        systemctl enable --now install-jumpbox-ssh-key.service
      SHELL

      # Install networkd-dispatcher actions.
      #
      # This is necessary due to the aforementioned limitation where we can't
      # set a static IP on the private network interface.
      opts[:name] != "jumpbox" and node.vm.provision "shell", inline: <<-SHELL
        set -eux

        rsync -a --no-o --no-g \
            /vagrant/files/etc/networkd-dispatcher/ /etc/networkd-dispatcher/
        systemctl start networkd-dispatcher.service
        networkctl reconfigure eth1
      SHELL

      # Install service for updating /etc/hosts and routes periodically.
      # Again, because we can't set a static IP... would be nice!
      node.vm.provision "shell", inline: <<-SHELL
        set -eux
        rsync -a --no-o --no-g /vagrant/files/usr/local/bin/ /usr/local/bin/
        cp /vagrant/files/etc/systemd/system/update-hosts.{service,timer} \
          /etc/systemd/system/
        systemctl daemon-reload
        systemctl enable --now update-hosts.timer
      SHELL


      # -- Kubernetes ----------------------------------------------------------

      # Set up the Kubernetes apt repository on every box
      false and node.vm.provision "shell", inline: <<-SHELL
        set -eux

        apt-get update
        apt-get install -y \
          apt-transport-https \
          ca-certificates \
          curl \
          gnupg

        mkdir -p -m 755 /etc/apt/keyrings
        curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.33/deb/Release.key | \
          gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
        chmod 644 /etc/apt/keyrings/kubernetes-apt-keyring.gpg

        echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg]' \
          'https://pkgs.k8s.io/core:/stable:/v1.33/deb/ /' \
          > /etc/apt/sources.list.d/kubernetes.list
        chmod 644 /etc/apt/sources.list.d/kubernetes.list
      SHELL

      # Install kubectl on the jumpbox
      false and opts[:name] == "jumpbox" and node.vm.provision "shell", inline: <<-SHELL
        set -eux
        apt-get update
        apt-get install -y kubectl
      SHELL
    end
  end
end
