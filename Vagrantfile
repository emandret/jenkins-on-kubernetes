Vagrant.configure("2") do |config|

  config.vm.box = "hashicorp/bionic64"

  config.vm.provider "virtualbox" do |v, options|
    options.vm.synced_folder "/home/playbooks", "/nfs/playbooks", type: "nfs"
  end

  config.vm.define "kubemaster" do |kubemaster|
    kubemaster.vm.hostname = "kubemaster"
    kubemaster.vm.provider "virtualbox" do |v, options|

      v.memory = 2048
      v.cpus = 2
      options.vm.network "private_network"

      options.vm.provision "shell", inline: <<-SHELL
      ip link set dev eth1 up
      ip addr flush dev eth1
      ip addr add 192.168.1.0/24 dev eth1
      ip route flush dev eth1
      ip route add 192.168.1.0/24 dev eth1
      ip route add default via 10.0.2.1
      SHELL
    end
  end

  (1..3).each do |i|
    config.vm.define "worker-#{i}" do |worker|
      worker.vm.hostname = "worker-#{i}"
      worker.vm.provider "virtualbox" do |v, options|

        v.memory = 1024
        v.cpus = 1
        options.vm.network "private_network"

        options.vm.provision "shell", inline: <<-SHELL
        ip link set dev eth1 up
        ip addr flush dev eth1
        ip addr add 192.168.1.#{i}/24 dev eth1
        ip route flush dev eth1
        ip route add 192.168.1.0/24 dev eth1
        ip route add default via 10.0.2.1
        SHELL
      end
    end
  end

end
