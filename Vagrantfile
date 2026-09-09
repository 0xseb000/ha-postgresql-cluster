Vagrant.configure("2") do |config|

    config.vm.box = "bento/ubuntu-24.04"

    machines = {
    "ha"    => { ip: "10.255.255.10", memory: 2048, cpus: 2, prl_name: "haproxy" },
    "pg1" => { ip: "10.255.255.21", memory: 2048, cpus: 1 },
    "pg2" => { ip: "10.255.255.22", memory: 2048, cpus: 1 },
    "pg3" => { ip: "10.255.255.23", memory: 2048, cpus: 1 },
    "etcd"  => { ip: "10.255.255.30", memory: 1024, cpus: 1 }
    }

    machines.each_with_index do |(name, opts), i|
        config.vm.define name do |node|
            node.vm.hostname = name
            node.vm.network "private_network", ip: opts[:ip]

            node.vm.provider "parallels" do |prl|
                prl.name   = opts.fetch(:prl_name, name)
                prl.memory = opts[:memory]
                prl.cpus   = opts[:cpus]
            end

            # once arrive at the last machine, run ansible
            if i == machines.size - 1
                node.vm.provision "ansible" do |ansible|
                    ansible.playbook = "playbook/site.yml"
                    ansible.inventory_path = "inventory/hosts.yml"
                    ansible.limit = "all"
                end
            end
        end
    end
end