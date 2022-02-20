Vagrant.configure("2") do |config|

  config.vm.define "k8s-master" do |kubemaster|
    kubemaster.vm.hostname = "k8s-master"
    kubemaster.vm.box = "bento/ubuntu-20.04"
    kubemaster.vm.network "private_network", ip: "192.168.56.80"
    kubemaster.vm.provider "virtualbox" do |vb|
      vb.memory = "2048"
      vb.cpus = 2
    end
  end

  (1..2).each do |i|
    config.vm.define "k8s-worker#{i}" do |worker|
      worker.vm.hostname = "k8s-worker#{i}"
      worker.vm.box = "bento/ubuntu-20.04"
      worker.vm.network "private_network", ip: "192.168.56.8#{i}"
      worker.vm.provider "virtualbox" do |vb|
        vb.memory = "2048"
        vb.cpus = 2
      end
    end
  end
end
