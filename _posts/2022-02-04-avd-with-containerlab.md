---
layout: post
title: Build Containerlab topology from AVD
date: 2022-02-04 15:09 +0100
---

## Overview

Until recently, if we want to do networking lab, we had option to play with GNS3 or EVE-NG with Virtual Machine for Network OS. But 2021 has seen the rise of [Containerlab](https://containerlab.srlinux.dev/) which gives us ability to play Lab As Code as you can define your own topology in a YAML file.

Arista cEOS is [natively supported](https://containerlab.srlinux.dev/manual/kinds/ceos/) by the tool and gives you opportunity to build lab easily from any type of tool in charge of your design. __Big thanks you to Nokia's team__ for making this tool fully open-source and with support for all NOS.

In this article, we will see how to build a [containerlab topology](https://containerlab.srlinux.dev/manual/topo-def-file/) from an [Arista Validated Design collection](https://avd.sh/en/latest/) for ansible where you define your EVPN topology.

![](https://avd.sh/en/latest/media/avd-logo.png)

Yes, with AVD, you can easily build any type of EVPN/VXLAN fabric, but at some point it is nice to test configuration and building topology on the fly would make design easier.

![](/assets/img/avd-to-containerlab/avd-example-topo.png)

Of course, you can also find some __containerlab__ examples with [Arista cEOS on github](https://github.com/arista-netdevops-community/avd-quickstart-containerlab), but here we are going to talk about generating lab from Ansible.

## AVD to design an EVPN fabric

This guide is not a detailed guide to setup Arista AVD collection. For this type of document, you should visit [specific documentation](https://avd.sh/en/latest/docs/installation/collection-installation.html) for that. Here we will build a basic L3LS EVPN fabric based on 2 spines, 4 groups of 2 vteps and some L2 edge switches.

![AVD Topology](/assets/img/avd-to-containerlab/avd-topo.png)

### Install environment

Here we are talking about [python](https://www.python.org/downloads/) for ansible and [Golang](https://go.dev/doc/install) for Containerlab. And obviously, you need a docker engine to bring up lab at the end.

```bash
# Download AVD requirements
$ curl -fsSL https://raw.githubusercontent.com/aristanetworks/ansible-avd/devel/ansible_collections/arista/avd/requirements.txt
$ pip3 install -r requirements.txt

# Install ansible
$ pip3 install ansible-core
```

Once ansible is installed, you can use galaxy to install AVD and [avd_to_containerlab](https://github.com/titom73/ansible-inetsix/tree/master/ansible_collections/titom73/avd_tools/roles/eos_designs_to_containerlab) role:

```bash
$ ansible-galaxy collection install arista.avd
$ ansible-galaxy collection install titom73.avd_tools
```

Now we can install [Containerlab](https://containerlab.srlinux.dev/install/):

```bash
$ bash -c "$(curl -sL https://get-clab.srlinux.dev)"
```

Since we are here, let's download a cEOS version from Arista website. For that, you can use [eos-download](https://github.com/titom73/arista-downloader) as explained in this [post](). And finally, install it on your Machine.

```bash
$ docker import cEOS-lab.tar.xz ceosimage:<your-ceos-version>
```

Now we are ready to go to the build phase.

### Ansible configuration

Topology file is provided below and will be the key to build containerlab topology

- Spine topology:

```yaml
spine:
  defaults:
    platform: vEOS-LAB
    bgp_as: 65001
    loopback_ipv4_pool: 192.168.1.0/24
    bgp_defaults: '{{bgp_defaults_veos}}'
    mlag_peer_ipv4_pool: 172.31.253.0/31
    mlag_peer_l3_ipv4_pool: 172.31.253.2/31
  nodes:
    ceos-spine1:
      id: 1
      mgmt_ip: 10.73.255.101/24
      mac_address: '0c:1d:c0:1d:62:01'
    ceos-spine2:
      id: 2
      mgmt_ip: 10.73.255.102/24
      mac_address: '0c:1d:c0:1d:62:02'
```

- vTEP (aka L3LEAF) topology

```yaml
l3leaf:
  defaults:
    loopback_ipv4_pool: 192.168.255.0/24
    loopback_ipv4_offset: 2
    vtep_loopback_ipv4_pool: 192.168.254.0/24
    uplink_interfaces: ['Ethernet1', 'Ethernet2']
    uplink_switches: ['ceos-spine1', 'ceos-spine2']
    uplink_ipv4_pool: 172.31.255.0/24
    evpn_route_servers: ['ceos-spine1', 'ceos-spine2']
    mlag_peer_ipv4_pool: 172.31.253.0/31
    mlag_peer_l3_ipv4_pool: 172.31.253.2/31
    mlag_interfaces: '{{ mlag_interfaces_veos }}'
    virtual_router_mac_address: 00:1c:73:00:dc:01
    bgp_defaults: '{{ bgp_defaults_veos }}'
    spanning_tree_priority: 4096
    spanning_tree_mode: mstp
  node_groups:
    ceos_leaf1:
      bgp_as: 65101
      filter:
        tenants: [ 'Tenant_A', 'Tenant_B' ]
        tags: [ 'POD01', 'DC1' ]
      nodes:
        ceos-leaf1a:
          id: 11
          mgmt_ip: 10.73.255.111/24
          uplink_switch_interfaces: [Ethernet1, Ethernet1]
          mac_address: '0c:1d:c0:1d:62:11'
        ceos-leaf1b:
          id: 12
          mgmt_ip: 10.73.255.112/24
          uplink_switch_interfaces: [Ethernet2, Ethernet2]
          mac_address: '0c:1d:c0:1d:62:12'

# [ Output ommitted for readability]
# ...
```

- L2 Switch (aka L2LEAF) topology

```yaml
edge:
  defaults:
    uplink_switches: ['ceos-leaf1a', 'ceos-leaf1b']
    uplink_interfaces: ['Ethernet1', 'Ethernet2']
    mlag_peer_ipv4_pool: 172.31.253.0/31
    mlag_peer_l3_ipv4_pool: 172.31.253.2/31
  node_groups:
    ceos_edge_leaf1:
      uplink_switches: [ ceos-leaf1a, ceos-leaf1b ]
      filter:
        tenants: [ 'Tenant_A', 'Tenant_B' ]
        tags: [ 'POD01', 'DC1' ]
      nodes:
        ceos-agg01:
          id: 21
          mgmt_ip: 10.73.255.121/24
          mac_address: '0c:1d:c0:1d:62:21'
          uplink_switch_interfaces: [ Ethernet5, Ethernet5 ]

# [ Output ommitted for readability]
# ...
```

For a complete list of options, it is better to read the [full documentation of AVD](https://avd.sh/en/latest/roles/eos_designs/doc/fabric-topology.html).

Also, full inventory is available on [github](https://www.github.com/titom73/avd-to-containerlab/). So you can just clone and play.

### Playbook

After inventory has been created, we can create a playbook. This one will be fairly easy and will do 3 things:

- Transform abstracted topology into YAML EOS configuration file
- Generate EOS Cli configuration file
- Generate a containerlab topology loading AVD generated configuration files.

So here we go:

```yaml
# playbook/build.yml
---
- name: Build Switch configuration
  hosts: [all]
  tags:
    - avd
  collections:
    - arista.avd
  tasks:
    - name: generate intended variables
      tags: [build]
      import_role:
        name: eos_designs
    - name: generate device intended config and documentation
      tags: [build]
      import_role:
        name: eos_cli_config_gen

- name: Build containerlab topology
  hosts: [all]
  connection: local
  gather_facts: false
  collections:
    - arista.avd
    - titom73.avd_tools
  tasks:
    - name: 'Build Containerlab Topology'
      tags:
        - containerlab
      import_role:
        name: inetsix.avd_tools.eos_designs_to_containerlab
      vars:
        mgmt_network_v4: 10.73.255.0/24
        ceos_version: arista/ceos:4.27.1F
        eapi_base: 8000
```

As you can see, [`eos_designs_to_containerlab`](https://github.com/titom73/ansible-inetsix/tree/master/ansible_collections/titom73/avd_tools/roles/eos_designs_to_containerlab) role needs some elements to generate a full configuration:

- `mgmt_network_v4`: Management network to configure on docker side. It __must__ be the same as the one used in AVD
- `eapi_base`: Base port for eAPI to expose eAPI service to the network outside of docker.
- `ceos_version`: Which version of __cEOS__ to use.

## Make the magic happens


```bash
$ ansible-playbook playbooks/containerlab-build.yml -i containerlab/inventory.yml
[...]
===============================================================================
arista.avd.eos_cli_config_gen : Generate content of device documentation ---------- 4.61s
arista.avd.eos_cli_config_gen : Generate eos intended configuration --------------- 3.30s
arista.avd.eos_designs : Generate device configuration in structured format ------- 1.75s
arista.avd.eos_cli_config_gen : Generate TOC for device documentation ------------- 0.92s
arista.avd.eos_designs : Write device structured configuration to YAML file ------- 0.85s
arista.avd.eos_designs : Generate fabric documentation in Markdown Format. -------- 0.72s
arista.avd.eos_cli_config_gen : Create required output directories if not present - 0.61s
arista.avd.eos_designs : Set AVD facts -------------------------------------------- 0.58s
arista.avd.eos_cli_config_gen : Store checksum of existing device documentation --- 0.50s
arista.avd.eos_designs : Generate TOC for fabric documentation -------------------- 0.47s
arista.avd.eos_designs : Generate fabric topology in csv format. ------------------ 0.46s
inetsix.avd_tools.eos_designs_to_containerlab : Generate containerlab topology ---- 0.45s
arista.avd.eos_designs : Create required output directories if not present -------- 0.44s
arista.avd.eos_designs : Set AVD topology facts ----------------------------------- 0.28s
arista.avd.eos_designs : Store checksum of existing fabric documentation ---------- 0.26s
Playbook run took 0 days, 0 hours, 0 minutes, 17 seconds
```

So now you can look at files generated:

- EOS configuration files under `intended/configs`
- Containerlab topology in `containerlab/containerlab.yml`

```yaml
---
name: ceos_fabric

mgmt:
  network: 'mgmt_ceos_fabric'
  ipv4_subnet: 10.73.255.0/24

topology:
  kinds:
    ceos:
      image: arista/ceos:4.27.1F
  nodes:
    ceos-agg01:
      image: arista/ceos:4.27.1F
      mgmt_ipv4: 10.73.255.121
      kind: ceos
      startup-config: <path>/intended/configs/ceos-agg01.cfg
      ports:
      - 8021:443/tcp
      env:
        TMODE: lacp
# [...]
 links:
    - endpoints: ["ceos-agg01:eth1", "ceos-leaf1a:eth5"]
# [...]
```

And you can start containerlab:

```bash
$ sudo containerlab deploy --topo inventories/containerlab/containerlabs.yml
[sudo] password for tom:

INFO[0000] Parsing & checking topology file: containerlabs.yml
INFO[0000] Creating lab directory: /home/tom/arista-\
    avd/avd-lab-validation/clab-ceos_fabric
INFO[0000] Creating docker network: Name='mgmt_ceos_fabric', \
    IPv4Subnet='10.73.255.0/24', IPv6Subnet='', MTU='1500'
INFO[0000] config file '/home/tom/arista-avd/avd-lab-validation/\
    clab-ceos_fabric/ceos-agg01/flash/startup-config' for node \
    'ceos-agg01' already exists and will not be generated/reset
INFO[0000] Creating container: ceos-agg01
[...]
INFO[0004] Creating virtual wire: ceos-leaf3a:eth3 <--> ceos-leaf4a:eth3
INFO[0005] Running postdeploy actions for Arista cEOS 'ceos-leaf2a' node
INFO[0005] Running postdeploy actions for Arista cEOS 'ceos-bl01a' node
INFO[0005] Running postdeploy actions for Arista cEOS 'ceos-leaf3a' node
INFO[0005] Running postdeploy actions for Arista cEOS 'ceos-leaf4a' node
INFO[0005] Running postdeploy actions for Arista cEOS 'ceos-agg02' node
INFO[0005] Running postdeploy actions for Arista cEOS 'ceos-leaf1a' node
INFO[0005] Running postdeploy actions for Arista cEOS 'ceos-leaf2b' node
INFO[0005] Running postdeploy actions for Arista cEOS 'ceos-leaf1b' node
INFO[0005] Running postdeploy actions for Arista cEOS 'ceos-spine1' node
INFO[0005] Running postdeploy actions for Arista cEOS 'ceos-spine2' node
INFO[0005] Running postdeploy actions for Arista cEOS 'ceos-bl01b' node
INFO[0005] Running postdeploy actions for Arista cEOS 'ceos-agg01' node
```

Once it is over, you should see output below. It gives you some information such as `kind`, `version` and `management 0` IP address.

```bash
INFO[0143] 🎉 New containerlab version 0.23.0 is available! Release notes: https://containerlab.srlinux.dev/rn/0.23/
Run 'containerlab version upgrade' to upgrade or go check other installation options at https://containerlab.srlinux.dev/install/
+----+------------------------------+--------------+---------------------+------+---------+------------------+--------------+
| #  |             Name             | Container ID |        Image        | Kind |  State  |   IPv4 Address   | IPv6 Address |
+----+------------------------------+--------------+---------------------+------+---------+------------------+--------------+
|  1 | clab-ceos_fabric-ceos-agg01  | 37973cf400d7 | arista/ceos:4.27.1F | ceos | running | 10.73.255.121/24 | N/A          |
|  2 | clab-ceos_fabric-ceos-agg02  | cb37f8b37e0f | arista/ceos:4.27.1F | ceos | running | 10.73.255.122/24 | N/A          |
|  3 | clab-ceos_fabric-ceos-bl01a  | 9ba07822490b | arista/ceos:4.27.1F | ceos | running | 10.73.255.115/24 | N/A          |
|  4 | clab-ceos_fabric-ceos-bl01b  | 40eaf38dd58c | arista/ceos:4.27.1F | ceos | running | 10.73.255.116/24 | N/A          |
|  5 | clab-ceos_fabric-ceos-leaf1a | f4ac53c2ae4d | arista/ceos:4.27.1F | ceos | running | 10.73.255.111/24 | N/A          |
|  6 | clab-ceos_fabric-ceos-leaf1b | c03adbff83af | arista/ceos:4.27.1F | ceos | running | 10.73.255.112/24 | N/A          |
|  7 | clab-ceos_fabric-ceos-leaf2a | b59260c441a0 | arista/ceos:4.27.1F | ceos | running | 10.73.255.113/24 | N/A          |
|  8 | clab-ceos_fabric-ceos-leaf2b | 5514405b6429 | arista/ceos:4.27.1F | ceos | running | 10.73.255.114/24 | N/A          |
|  9 | clab-ceos_fabric-ceos-leaf3a | 87e22287c2f2 | arista/ceos:4.27.1F | ceos | running | 10.73.255.117/24 | N/A          |
| 10 | clab-ceos_fabric-ceos-leaf4a | 878b75601225 | arista/ceos:4.27.1F | ceos | running | 10.73.255.118/24 | N/A          |
| 11 | clab-ceos_fabric-ceos-spine1 | 7d10d85458e0 | arista/ceos:4.27.1F | ceos | running | 10.73.255.101/24 | N/A          |
| 12 | clab-ceos_fabric-ceos-spine2 | 7e0effa1ccd6 | arista/ceos:4.27.1F | ceos | running | 10.73.255.102/24 | N/A          |
+----+------------------------------+--------------+---------------------+------+---------+------------------+--------------+

```

Also, you can look at docker directly. This one is interesting to ge eAPI port mapping:

```bash
docker ps
CONTAINER ID   IMAGE                  COMMAND                  CREATED         STATUS         PORTS                                       NAMES
87e22287c2f2   arista/ceos:4.27.1F    "/sbin/init systemd.…"   2 minutes ago   Up 2 minutes   0.0.0.0:8015->443/tcp, :::8015->443 tcp     clab-ceos_fabric-ceos-leaf3a
f4ac53c2ae4d   arista/ceos:4.27.1F    "/sbin/init systemd.…"   2 minutes ago   Up 2 minutes   0.0.0.0:8011->443/tcp, :::8011->443/tcp     clab-ceos_fabric-ceos-leaf1a
5514405b6429   arista/ceos:4.27.1F    "/sbin/init systemd.…"   2 minutes ago   Up 2 minutes   0.0.0.0:8014->443/tcp, :::8014->443/tcp     clab-ceos_fabric-ceos-leaf2b
40eaf38dd58c   arista/ceos:4.27.1F    "/sbin/init systemd.…"   2 minutes ago   Up 2 minutes   0.0.0.0:8018->443/tcp, :::8018->443/tcp     clab-ceos_fabric-ceos-bl01b
b59260c441a0   arista/ceos:4.27.1F    "/sbin/init systemd.…"   2 minutes ago   Up 2 minutes   0.0.0.0:8013->443/tcp, :::8013->443/tcp     clab-ceos_fabric-ceos-leaf2a
cb37f8b37e0f   arista/ceos:4.27.1F    "/sbin/init systemd.…"   2 minutes ago   Up 2 minutes   0.0.0.0:8022->443/tcp, :::8022->443/tcp     clab-ceos_fabric-ceos-agg02
c03adbff83af   arista/ceos:4.27.1F    "/sbin/init systemd.…"   2 minutes ago   Up 2 minutes   0.0.0.0:8012->443/tcp, :::8012->443/tcp     clab-ceos_fabric-ceos-leaf1b
7d10d85458e0   arista/ceos:4.27.1F    "/sbin/init systemd.…"   2 minutes ago   Up 2 minutes   0.0.0.0:8001->443/tcp, :::8001->443/tcp     clab-ceos_fabric-ceos-spine1
878b75601225   arista/ceos:4.27.1F    "/sbin/init systemd.…"   2 minutes ago   Up 2 minutes   0.0.0.0:8016->443/tcp, :::8016->443/tcp     clab-ceos_fabric-ceos-leaf4a
7e0effa1ccd6   arista/ceos:4.27.1F    "/sbin/init systemd.…"   2 minutes ago   Up 2 minutes   0.0.0.0:8002->443/tcp, :::8002->443/tcp     clab-ceos_fabric-ceos-spine2
37973cf400d7   arista/ceos:4.27.1F    "/sbin/init systemd.…"   2 minutes ago   Up 2 minutes   0.0.0.0:8021->443/tcp, :::8021->443/tcp     clab-ceos_fabric-ceos-agg01
9ba07822490b   arista/ceos:4.27.1F    "/sbin/init systemd.…"   2 minutes ago   Up 2 minutes   0.0.0.0:8017->443/tcp, :::8017->443/tcp     clab-ceos_fabric-ceos-bl01a
```

And you can connect to your EOS cli directly from host with docker command:

```bash
docker exec -it clab-ceos_fabric-ceos-spine1 Cli
ceos-spine1>
ceos-spine1>en

ceos-spine1#show lldp neighbors
Last table change time   : 0:12:19 ago
Number of table inserts  : 20
Number of table deletes  : 1
Number of table drops    : 0
Number of table age-outs : 1

Port          Neighbor Device ID       Neighbor Port ID    TTL
---------- ------------------------ ---------------------- ---
Et1           ceos-leaf1a              Ethernet1           120
Et2           ceos-leaf1b              Ethernet1           120
Et3           ceos-leaf2a              Ethernet1           120
Et4           ceos-leaf2b              Ethernet1           120
Et5           ceos-bl01a               Ethernet1           120
Et6           ceos-bl01b               Ethernet1           120
Et7           ceos-leaf3a              Ethernet1           120
Et8           ceos-leaf4a              Ethernet1           120

ceos-spine1#show bgp evpn summary
BGP summary information for VRF default
Router identifier 192.168.1.1, local AS number 65001
Neighbor Status Codes: m - Under maintenance
  Description  Neighbor         V AS    MsgRcvd MsgSent InQ OutQ  Up/Down State  PfxRcd PfxAcc
  ceos-leaf1a  192.168.255.13   4 65101      46      42   0    0 00:13:01 Estab  10     10
  ceos-leaf1b  192.168.255.14   4 65101      46      42   0    0 00:13:01 Estab  10     10
  ceos-leaf2a  192.168.255.15   4 65102      51      41   0    0 00:13:07 Estab  10     10
  ceos-leaf2b  192.168.255.16   4 65102      52      42   0    0 00:13:08 Estab  10     10
  ceos-leaf3a  192.168.255.17   4 65103      52      44   0    0 00:13:16 Estab  5      5
  ceos-leaf4a  192.168.255.18   4 65104      53      48   0    0 00:13:28 Estab  5      5
  ceos-bl01a   192.168.255.19   4 65105      52      48   0    0 00:13:18 Estab  4      4
  ceos-bl01b   192.168.255.20   4 65105      54      48   0    0 00:13:18 Estab  4      4
```

If you change your AVD parameter, your lab will be fully rebuild and will be ready for testing. You just have to destroy active topology before running again the playbook.

## Resources

- [Containerlab](https://containerlab.srlinux.dev) / [User Guide](https://containerlab.srlinux.dev/manual/topo-def-file/)
- Containerlab [examples](https://containerlab.srlinux.dev/lab-examples/lab-examples/)
- Arista [EOS AVD Topology examples](https://github.com/arista-netdevops-community/avd-quickstart-containerlab)
- Arista AVD ansible Collection: [Documentation](https://www.avd.sh/) / [Repository](https://github.com/aristanetworks/ansible-avd/)
- Collection to [map AVD with Containerlab](https://github.com/titom73/ansible-inetsix/tree/master/ansible_collections/titom73/avd_tools/roles/eos_designs_to_containerlab)
- [User journey & experiences](https://juliopdx.com/2021/12/10/my-journey-and-experience-with-containerlab/) with Containerlab by [@Julio](https://twitter.com/Julio_PDX)
