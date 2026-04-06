# ansible proxmox role

This role installs and configures network, snmpd, packages, sets the hostname and system time for a proxmox 9.X host or cluster.

## Requirements

Proxmox install of 9.1 or greater.

## Dependencies

- `community.general.timezone`

## Use the role

You can use this role through Ansible `requirements.yaml` or Git submodules to install the role in your playbook's `roles/` directory.

### Ansible requirements.yaml

Ansible can also pull the role from GitLab directly. Add this to your `requirements.yaml` file at the root of your playbook:

```yaml
roles:
  - name: anro_proxmox
    src: git+https://github.com/jmcnab006/anro_proxmox
    version: main
```

Use the `ansible-galaxy` command to install all roles defined in your `requirements.yaml`:

```command
ansible-galaxy install -r requirements.yaml
```

Use the name of the role directory in the `roles:` list to use it in your playbook. And specify
`vars:`  in the group_vars or in the role definition. 

```yaml
- name: ROLES | Run proxmox role on all hosts. 
  hosts: all
  roles:
    - anro_proxmox
      vars:
        # sets time server definition
        anro_proxmox_timeserver: pool.ntp.org

        # Check and apply updates at runtime
        anro_proxmox_apply_updates: true

        # List of domains to add to the /etc/hosts file
        anro_proxmox_hosts_domains:
          - pve.bridgelinetech.com

        # Set our clustering settings. Cluster should be created in the ui with a single network. 
        anro_proxmox_cluster_enabled: true
        anro_proxmox_cluster_join: true

        # name set in the UI
        anro_proxmox_cluster_name: "pve-cluster01"

        # Fingerprint from UI 
        anro_proxmox_cluster_fingerprint: "5D:88:C0:C8:33:24:20:CC:A0:FF:22:F9:CD:A7:8E:D9:34:92:9C:7E:80:97:F5:92:72:E6:6E:09:40:04:F9:92"
        anro_proxmox_cluster_master: 172.16.0.101

        # Install CEPH, this does not do any configuration just ensures the packages are installed. Configure through the ui.
        # you can create disks easily with perl and pveceph command:
        # for example for NVME disks 0-7 this creates new OSDs of device class NVME. 
        # perl -e 'for (0 .. 7 ){ $cmd = "/usr/bin/pveceph osd create /dev/nvme" . $_ . "n1 -crush-device-class nvme"; print $cmd . "\n"; $x = `$cmd`; print $x; }'
        anro_proxmox_ceph_enabled: true

        # Network that is defined for pve if clustering is not used the default should be fine.
        anro_proxmox_network: 172.16.0.0
        anro_proxmox_network_mask: 24

        # Interface network configuration, this is applied directly to the /etc/network/interfaces
        anro_proxmox_network_interfaces:
          # enable our management interface
          - name: eno5
            address: "{{ ansible_host }}"
            netmask: 255.255.255.0
            gateway: 10.255.255.1
            family: inet
            method: static
            comment: "out of band management network"

          # Enable our interfaces to be used by the virtual interfaces and OVS bridge
          - name: eno6
          - name: eno7
          - name: eno8

          # Define a virtual interface for pve cluster/ceph interface
          - name: vmk0
            method: static
            address: "{{ anro_proxmox_pve_network_address }}/{{anro_proxmox_network_mask}}"
            ovs_type: OVSIntPort
            ovs_bridge: vmbr0
            ovs_options: tag=4065
            comment: "pve cluster network"

          # Define a bond to assign to the bridge
          - name: bond0
            ovs_bonds: eno7 eno8
            ovs_type: OVSBond
            ovs_bridge: vmbr0
            ovs_options: lacp=active bond_mode=balance-tcp
            comment: "virtual_switch0 bond"

          # Define our backplane virtual switch
          - name: vmbr0
            ovs_type: OVSBridge
            ovs_ports: bond0 vmk0
            comment: "virtual_switch0 bridge"

```

## Role variables

The role is designed to run defaults the majority of the time, but has some customizations to accommodate different environments.

Refer to [argument specifications](meta/argument_specs.yaml) for detailed information on all role variables.
