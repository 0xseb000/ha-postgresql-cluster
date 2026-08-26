Vagrant.configure("2") do |config|

    config.vm.box = "bento/ubuntu-24.04"

    # HA Proxy Node
    config.vm.define "ha" do |ha|
        ha.vm.hostname = "ha"
        ha.vm.network "private_network", ip: "10.255.255.10"
        ha.vm.provider "parallels" do |prl|
            prl.name = "haproxy"
            prl.memory = 2048
            prl.cpus = 2
        end
    end

    # DB Cluster
    (1..3).each do |i|
        config.vm.define "db-0#{i}" do |nd|
            nd.vm.hostname = "db-0#{i}"
            nd.vm.network "private_network", ip: "10.255.255.2#{i}"
            nd.vm.provider "parallels" do |prl|
                prl.name = "db-#{i}"
                prl.memory = 2048
                prl.cpus = 1
            end
        end
    end

    # etcd Node
    config.vm.define "etcd" do |nd|
        nd.vm.hostname = "etcd"
        nd.vm.network "private_network", ip: "10.255.255.30"
        nd.vm.provider "parallels" do |prl|
            prl.name = "etcd"
            prl.memory = 2048
            prl.cpus = 1
        end
    end

    # starts ansible playbook | run with --no-provision until ansible is configured
    config.vm.provision "ansible" do |ansible|
        ansible.playbook = "playbook/site.yml"
        ansible.inventory_path = "inventory/hosts.yml"
        ansible.limit = "all"
    end

end