require "yaml"
require "etc"
require "json"

KEY_DIR = "/m/salt-keys"

conf = YAML.load_file("./saltnetcfg.yaml")
minions = conf["minions"]
bridge = conf["bridge"]
master_ip = conf["master_ip"]

# handle_system
# config - Vagrant configure object
# host - Hash of host config data
# master_ip - Address of master for this host
# bridge - Network bridge device, if public network
def handle_host(config, host, master_ip, bridge)
  config.vm.define host["name"] do |node|
    node.vm.provider "virtualbox" do |vb|
      vb.name = host["name"]
    end
    node.vm.box = host["box"]
    node.vm.hostname = host["name"]
    if host["private_net"]
      node.vm.network "private_network", ip: host["ip"]
    else
      node.vm.network "public_network", bridge: bridge, ip: host["ip"]
    end
    node.vm.provision :salt do |salt|
      # The Salt minion_id config is currently ignored and Salt bootstrap
      # uses one of the hostnames that it finds. This is not necessarily
      # the hostnam set here. It worked on the test box that was used
      # originally that did not have a hostname previously set.  But,
      # failed on public boxes that already had set a separate hostname.
      # For now, set it in bootstrap_options.
      #salt.minion_id = "testminionname"
      salt.install_type = "stable"
      bootstrap_options = "-D -A #{master_ip} -i #{host["name"]}"
      salt.minion_key = File.join(KEY_DIR, host["name"]) + ".pem"
      salt.minion_pub = File.join(KEY_DIR, host["name"]) + ".pub"
      master_cfg = {}
      if host["minions"]
        # There are relative minions of this node, so set up salt-master and
        # salt-minion. Preseed this nodes master with keys for those minions.
        minion_keys = {}
        host["minions"].each do |minion|
          minion_name = minion["name"]
          minion_keys[minion_name] = File.join(KEY_DIR, "#{minion_name}.pub")
        end
        master_cfg["syndic_master"] = master_ip
        salt.seed_master = minion_keys
        salt.install_master = true
        salt.install_syndic = true
      end
      if not master_cfg.empty?
          syndic_cfg = JSON.generate(master_cfg)
          bootstrap_options << " -J '" << syndic_cfg << "'"
      end
      salt.bootstrap_options = bootstrap_options
    end
  end
  if host["minions"]
    # Now set up any nodes listed as minions to this node.
    host["minions"].each do |minion|
      handle_host(config, minion, host["ip"], bridge)
    end
  end
end

def ensure_key(name)
  user = Etc.getlogin
  if not File.file?(File.join(KEY_DIR, "#{name}.pem"))
    system "salt-key -u #{user} --gen-keys=#{name} --gen-keys-dir=#{KEY_DIR}"
    puts "Generated new Salt keypair for '#{name}' minion."
  end
end

def do_host_keys(host)
  ensure_key(host["name"])
  if host["minions"]
    # Recurse through children.
    host["minions"].each do |minion|
      do_host_keys(minion)
    end
  end 
end

# Create missing minion keys first. There are two places to create if checked
# when used.
if (ARGV[0] == "up") || (ARGV[0] == "provision")
  minions.each do |minion|
    do_host_keys(minion)
  end
end

Vagrant.configure(2) do |config|
  config.vm.synced_folder ".", "/vagrant", disabled: true
  minions.each do |minion|
    handle_host(config, minion, master_ip, bridge)
  end
end


