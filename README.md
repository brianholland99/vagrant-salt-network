# Vagrant instructions for building a multi-tiered salt network with VMs

## Description

This uses existing boxes built for Vagrant. It installs the appropriate Salt
packages and configures the VMs according to the YAML configuration file. The
file can describe a multi-tiered Salt network using syndic nodes to relay
commands. This was intended to be used to test out Salt configuration, to
play with syndic, and to try and deal with OS variants in the Salt states.

The goal was to be totally repeatable setting up initial systems to test out
Salt states. Without resetting the systems back to initial state, remnants
can build up for various reasons and may not show deficiencies in the states.

## Configuration File Format

The "saltnetcfg.yaml" file describes the network layout. The root level
has the following attributes:

* bridge - Set to network device if bridged network is used (public_network).
* master_ip - Set to the address of the existing master for this set of nodes.
* minions - A hierarchical listing of minions.

Each minion has the following attributes:

* name - Currently used as the hostname and the minion_id.
* box - The VM box built for Vagrant to use as a base. This could be a
  locally built box or a public one.
* ip - The IP address of the minion. This is used for both setting IP address
  and as the master address for each direct minion.
* private_net: A boolean indicated whether the IP is to be treated
  as private (not bridged).
* minions - (Optional) A nest array of minions to this minion.

The minions can be nested multiple levels deep.

Example:

    bridge: eno2
    master_ip: 192.168.1.124
    minions:
      - name: minion01
        box: /m/boxes/vagrant-vb/min-centos7.2.box
        ip: 172.16.1.2
        private_net: true
        minions:
          - name: minion01-01
            box: /m/boxes/vagrant-vb/min-centos7.2.box
            ip: 172.16.1.4
            private_net: true
      - name: minion02
        box: /m/boxes/vagrant-vb/min-centos7.2.box
        ip: 172.16.1.3
        private_net: true
        master_ip: 

The above configuration describes three nodes to configure.  The `minion01`
and `minion02` nodes that are the two direct minions to the master declared
at the root level. The 'minion01-01` node is a minion of the `minion01` node.

The nesting will cause `minion01` will have Salt minion, master, and syndic
installed and configured and will relay commands and responses between its
minion and the master.

## Versions used for build to deploy

The following package versions have been used from building the boxes to
deploying via Vagrant.  Only Vagrant and Virtualbox is used directly. The
packer version listed is the one that built the box that was used successfully.

* OS Ubuntu 16.04 LTS
* Packer 0.10.2
* Vagrant 1.8.6
* Virtualbox 5.1.8

## Instructions

The following instructions should get a set of nodes up and running in
Virtualbox VMs. The minion keys are located in "KEY_DIR" directory defined
in the Vagrantfile. It is good to let them persist so that a minion gets the
same key if it is destroyed and remade. Otherwise the overall master for
top-level minions or masters left running when a minion's key is added
will not be happy.

The Vagrant file will generate new keys for nodes that do not yet have them.
If they are top-level minions, then the overall master needs to accept them or
have them set. If bringing up the complete set of nodes, all intermediate
masters will have the appropriate keys for their minions.

Altering the configuration file by adding nodes or changing hierarchy can cause
issues if parts are left up and the new nodes are brought up. Destroying the
complete set of nodes and bringing them up again should fix things.

### Steps

1. This assumes that a Salt master has been set up at the address indicated
   by the "master_ip" setting.
2. `vagrant up`
3. Once up, you should be able to issue a command on the overall master
   such as "salt '*' test.ping" and see the set of nodes.

Note: If deleting keys for top-level nodes from the KEY_DIR, remove them from
the overall master to prevent later minions with that name from being
denied.

### Cleanup

To clean up remnants, run the following command from this directory. This does
not clean up the KEY_DIR.

`./clean.sh`
