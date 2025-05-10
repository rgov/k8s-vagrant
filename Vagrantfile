Vagrant.configure("2") do |config|

  boxes = [
    { :name => "jumpbox", :role => :bastion,
      :cpus => 1, :memory => 512,  :disk => "10GB" },
    { :name => "server", :role => :control,
      :cpus => 1, :memory => 2048, :disk => "20GB" },
    { :name => "node-0",  :role => :worker, :podsubnet => "10.200.0.0/24",
      :cpus => 1, :memory => 2048, :disk => "20GB" },
    { :name => "node-1", :role => :worker, :podsubnet => "10.200.1.0/24",
      :cpus => 1, :memory => 2048, :disk => "20GB" },
  ]


  # Create a 'machines' file which contains a mapping of pod subnets to the node
  # hostname. At runtime, we'll create static routes.
  config.trigger.before :up do |trigger|
    trigger.name = "Output machines file"
    trigger.only_on = (boxes.find { |b| b[:role] == :bastion })[:name]  # first
    trigger.ruby do |env,machine|
      Dir.mkdir("shared") unless File.exist?("shared")
      File.open("shared/machines", "w") do |f|
        boxes.each do |b|
          next if not [:control, :worker].include? b[:role]
          f.puts "x.x.x.x #{b[:name]}.kubernetes.local #{b[:name]} #{b[:podsubnet]}"
        end
      end
    end
  end


  # Generate a certificate authority and certificates.
  #
  # /!\ WARNING: SECURITY IMPLICATIONS /!\
  # Because this directory is synced with each node as /vagrant, all of the
  # private keys are visible, including the certificate authority's signing key!
  remove_keys_after_copy = false  # false saves time but is more insecure
  config.trigger.before :up do |trigger|
    trigger.name = "Generate TLS certificates"
    trigger.ruby do |env,machine|
      next unless machine.name.to_s == "jumpbox"  # FIXME

      # Generate the certificate authority
      new_ca = false
      if not File.exist?("shared/ca.key")
        %x{openssl genrsa -out shared/ca.key 4096}
        new_ca = true
      end
      if new_ca or not File.exist?("shared/ca.crt")
        %x{openssl req -x509 -new -sha512 -noenc \
            -key shared/ca.key -days 3653 \
            -config files/ca.conf \
            -out shared/ca.crt}
        new_ca = true
      end

      # Generate and sign each of the various keys.
      #
      # TODO: Populate some of these from the list of boxes. However, this also
      # requires dynamically defining new sections in the ca.conf file.
      certs = [
        "admin", "node-0", "node-1",
        "kube-proxy", "kube-scheduler", "kube-controller-manager",
        "kube-api-server", "service-accounts",
      ]
      certs.each do |cert|
        new_key = false
        if not File.exist?("shared/#{cert}.key")
          %x{openssl genrsa -out shared/#{cert}.key 4096}
          new_key = true
        end
        if new_key or new_ca or not File.exist?("shared/#{cert}.crt")
          %x{openssl req -new -key shared/#{cert}.key -sha256 \
              -config files/ca.conf -section #{cert} \
              -out shared/#{cert}.csr}
          %x{openssl x509 -req -days 3653 -in shared/#{cert}.csr \
              -copy_extensions copyall \
              -sha256 -CA shared/ca.crt \
              -CAkey shared/ca.key \
              -CAcreateserial \
              -out shared/#{cert}.crt}
          File.unlink("shared/#{cert}.csr")
        end
      end
    end
  end

  boxes.each do |opts|
    config.vm.define opts[:name], primary: opts[:role] == :bastion do |node|
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

      # Disable swap
      node.vm.provision "shell", inline: <<-SHELL
        set -eux
        swapoff -a
        sed -i '/\\bswap\\b/ s/^/#/' /etc/fstab
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
      opts[:role] != :bastion and node.vm.provision "shell", inline: <<-SHELL
        set -eux
        HOSTNAME=$(hostname --short)
        FQDN=${HOSTNAME}.kubernetes.local
        sed -i "s/^127.0.2.1.*/127.0.2.1 ${FQDN} ${HOSTNAME}/" /etc/hosts
        test "$(hostname --fqdn)" = "$FQDN"
      SHELL

      # Generate an SSH key on the bastion hosts for the vagrant user
      opts[:role] == :bastion and
      node.vm.provision "shell", privileged: false, inline: <<-SHELL
        set -eux
        ssh-keygen -t rsa -f ~/.ssh/id_rsa -N ""
        cp ~/.ssh/id_rsa.pub /vagrant/shared/bastion_ssh_key.#{opts[:name]}.pub
      SHELL

      # All other boxes: Install the SSH keys.
      # FIXME: This assumes that all bastion hosts are provisioned first.
      opts[:role] != :bastion and
      node.vm.provision "shell", privileged: false, inline: <<-SHELL
        set -eux
        umask 077
        mkdir -p ~/.ssh || true
        cat /vagrant/shared/bastion_ssh_key.*.pub >> ~/.ssh/authorized_keys
      SHELL

      # Install networkd-dispatcher actions.
      #
      # This is necessary due to the aforementioned limitation where we can't
      # set a static IP on the private network interface.
      opts[:role] != :bastion and node.vm.provision "shell", inline: <<-SHELL
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
        cp /vagrant/files/etc/systemd/system/update-machines.{service,timer} \
          /etc/systemd/system/
        systemctl daemon-reload
        systemctl enable --now update-machines.timer
      SHELL


      # -- Certificate Authority -----------------------------------------------

      # Install server node keys and certificates for creating kubeconfigs later
      opts[:role] == :control and node.vm.provision "shell", inline: <<-SHELL
        set -eux
        cd /vagrant/shared/
        cp ca.key ca.crt \
          kube-api-server.key kube-api-server.crt \
          service-accounts.key service-accounts.crt \
          ~/

        if [ "#{remove_keys_after_copy}" = "true" ]; then
          rm -f /vagrant/shared/ca.key
        fi
      SHELL

      # Install worker node keys and certificates
      opts[:role] == :worker and node.vm.provision "shell", inline: <<-SHELL
        set -eux
        HOSTNAME="$(hostname --short)"
        mkdir -p /var/lib/kubelet
        cp /vagrant/shared/ca.crt /var/lib/kubelet/ca.crt
        cp /vagrant/shared/"$HOSTNAME".crt /var/lib/kubelet/kubelet.crt
        cp /vagrant/shared/"$HOSTNAME".key /var/lib/kubelet/kubelet.key

        if [ "#{remove_keys_after_copy}" = "true" ]; then
          rm -f /vagrant/shared/"$HOSTNAME".key
        fi
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
      false and opts[:role] == :jumpbox and node.vm.provision "shell", inline: <<-SHELL
        set -eux
        apt-get update
        apt-get install -y kubectl
      SHELL
    end
  end
end
