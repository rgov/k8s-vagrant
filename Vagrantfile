Vagrant.configure("2") do |config|

  boxes = [
    { :name => "jumpbox", :role => :bastion,
      :cpus => 1, :memory => 512,  :disk => "10GB" },
    { :name => "server", :role => :control,
      :cpus => 2, :memory => 2048, :disk => "20GB" },
    { :name => "worker-0",  :role => :worker,
      :cpus => 1, :memory => 2048, :disk => "20GB" },
    { :name => "worker-1", :role => :worker,
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

      # Enable IP forwarding
      opts[:role] != :bastion and node.vm.provision "shell", inline: <<-SHELL
        set -eux
        echo 'net.ipv4.ip_forward=1' > /etc/sysctl.d/99-ip-forward.conf
        sysctl --system
        test "$(cat /proc/sys/net/ipv4/ip_forward)" = "1"
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


      # -- Kubernetes ----------------------------------------------------------

      # Set up the Kubernetes apt repository on every box
      node.vm.provision "shell", inline: <<-SHELL
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

        echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg]' \
          'https://pkgs.k8s.io/core:/stable:/v1.33/deb/ /' \
          > /etc/apt/sources.list.d/kubernetes.list
      SHELL

      # Set up the CRI-O apt repository on every box
      node.vm.provision "shell", inline: <<-SHELL
        set -eux

        CRIO_VERSION=v1.32
        curl -fsSL \
          "https://download.opensuse.org/repositories/isv:/cri-o:/stable:/$CRIO_VERSION/deb/Release.key" | \
          gpg --dearmor -o /etc/apt/keyrings/cri-o-apt-keyring.gpg

        echo 'deb [signed-by=/etc/apt/keyrings/cri-o-apt-keyring.gpg]' \
          "https://download.opensuse.org/repositories/isv:/cri-o:/stable:/$CRIO_VERSION/deb/ /" \
          > /etc/apt/sources.list.d/cri-o.list
      SHELL

      # Install only kubectl on the bastion systems
      opts[:role] == :bastion and node.vm.provision "shell", inline: <<-SHELL
        set -eux
        apt-get update
        apt-get install -y kubectl
      SHELL

      # Install common dependencies on control and worker systems
      [:control, :worker].include?(opts[:role]) and
      node.vm.provision "shell", inline: <<-SHELL
        set -eux
        apt-get update
        apt-get install -y \
          cri-o \
          kubeadm \
          kubelet \
          kubectl \
          yq
        systemctl enable --now crio kubelet
      SHELL

      # Use kubeadm to initialize the cluster
      opts[:role] == :control and node.vm.provision "shell", inline: <<-SHELL
        set -eux

        FQDN=$(hostname --fqdn)
        ADDR=$(ip addr show dev eth1 | awk '/inet / {print $2}' | cut -d/ -f1)
        kubeadm init \
          --apiserver-advertise-address="$ADDR" \
          --control-plane-endpoint="${FQDN}:6443" \
          --cri-socket=unix:///var/run/crio/crio.sock \

        # Configure the Calico add-on
        export KUBECONFIG=/etc/kubernetes/admin.conf
        kubectl create -f \
          https://raw.githubusercontent.com/projectcalico/calico/v3.30.0/manifests/tigera-operator.yaml

        until kubectl get crd installations.operator.tigera.io; do
          sleep 1
        done 2>/dev/null

        kubectl create -f /vagrant/files/calico-custom.yaml
      SHELL

      # The control node generates a script for each worker to join the cluster
      boxes.select { |b| b[:role] == :worker }.each do |w|
        opts[:role] == :control and node.vm.provision "shell", inline: <<-SHELL
          SCRIPT=/vagrant/shared/cluster-join-#{w[:name]}.sh
          (echo "#!/bin/sh"; echo "set -eu") > "$SCRIPT"
          # TODO: Should wait for the server address to be known
          kubeadm token create --print-join-command >> "$SCRIPT"
          chmod +x "$SCRIPT"
        SHELL
      end

      # Run the cluster join script on each worker
      opts[:role] == :worker and node.vm.provision "shell", inline: <<-SHELL
        set -eux
        /vagrant/shared/cluster-join-#{opts[:name]}.sh
        rm -f /vagrant/shared/cluster-join-#{opts[:name]}.sh
      SHELL

    end
  end
end
