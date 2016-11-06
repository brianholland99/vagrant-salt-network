require "yaml"
require "fileutils"
require "etc"

TMP_DIR = "tmp-masterconfigs"
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
      # The Salt minion_id config is currently ignored and Salt uses hostname,
      # so make sure hostname above is set to the desired minion_id.
      # This was noticed when setting hostname to the FQDN and setting
      # minion_id to just the short name. 
      #salt.minion_id = "testminionname"
      salt.install_type = "stable"
      salt.bootstrap_options = "-A #{master_ip}"
      salt.minion_key = File.join(KEY_DIR, host["name"]) + ".pem"
      salt.minion_pub = File.join(KEY_DIR, host["name"]) + ".pub"
      if host["minions"]
        # There are relative minions of this node, so set up salt-master and
        # salt-minion. Preseed this nodes master with keys for those minions.
        minion_keys = {}
        host["minions"].each do |minion|
          minion_name = minion["name"]
          minion_keys[minion_name] = File.join(KEY_DIR, "#{minion_name}.pub")
        end
        # Make master config file to define "syndic_master" since neither
        # Vagrant Salt provisioner nor bootstrap-salt handle it directly.
        FileUtils.mkdir_p TMP_DIR
        File.open(File.join(TMP_DIR, host["name"]), "w") do |f|
          f.write("syndic_master: #{master_ip}\n")
        end
        #salt.master_config = "syndic_master/#{host['name']}"
        salt.master_config = File.join(TMP_DIR, host['name'])
        salt.seed_master = minion_keys
        salt.install_master = true
        salt.install_syndic = true
      end
    end
  end
  if host["minions"]
    # Now set up any nodes listed as minions to this node.
    host["minions"].each do |minion|
      handle_host(config, minion, host["ip"], bridge)
    end
  end
end

# Clear out previous syndic_master configs.
FileUtils.rm_rf TMP_DIR

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


