# SA_TO_SAs

## Section 1: Install Python3, pip3 and Ansible
```
mkdir Ansible_Workshop && cd Ansible_Workshop
sudo apt update
sudo apt install software-properties-common python3-pip python3-venv
```
Create a new python virtual environment

```
python3 -m venv .venv
```

Activate the virtual environment
```
source .venv/bin/activate
```
Now that we are inside the python environment we can install packages here that will not affect our system python environment. This allows you to use different versions of ansible or other python packages that can potentially conflict. This also helps make your Ansible playbooks portable.

Now run the below command in terminal to install the packages
```
pip3 install wheel ansible paramiko
```
Now that we have ansible installed we need to add a module that will help us connect and configure our topology
```
ansible-galaxy collection install cisco.ios arista.eos
```
We also need to setup the interface that is connected to the managment of the switches and router. Update the following file /etc/netplan/00-installer-config.yaml

```
sudo nano /etc/netplan/00-installer-config.yaml
```
```
# This is the network config written by 'subiquity'
network:
  ethernets:
    ens3:
      dhcp4: true
    ens4:
      dhcp4: false
      addresses:
        - 172.16.1.254/24
  version: 2
```

## Section 2: Creating the inventory yaml file
[Ansible inventory Documentation can be found here](https://docs.ansible.com/ansible/latest/user_guide/intro_inventory.html#) 
We will be constructing our inventory file with yaml format. Each pod has a bootstrap configuration that includes IP addressing to each node. Your control nodes (Jumpbox VM) has the /etc/hosts file built and each node has been assigned a hostname and an IP address.
Here are the groupings we will be building in our inventory file.
1. Routers
    *   podxr1
2. Core Switches
    *   podxsw1
    *   podxsw2
3. Access Switches
    *   podxsw3

The inventory file is what Ansible will use to connect to each of the managed hosts. We can assign each host to a group and each group to a parent group. Our file structure example is below along with the inventory.yml file.
```
inventory/
    inventory.yml
    group_vars/
        all/
            all.yml
    host_vars/
        podxr1/
        podxsw1/
        podxsw2/
        podxsw3/
```
inventory.yml - Our inventory file will use groupings of "all". "pod1" will be a child group of the parent "all". "routers", "core_switches" and "access_switches" will all be children of the parent group "pod1". 
```
---
all:
  children:
    pod1:
      children:
        routers:
          hosts:
            pod1r1:
              ansible_host: 172.16.1.1
              ansible_network_os: ios    
        core_switches:
          hosts:
            pod1sw1:
              ansible_host: 172.16.1.2
              ansible_network_os: eos
              ansible_become_method: enable
              ansible_become: true
            pod1sw2:
              ansible_host: 172.16.1.3
              ansible_network_os: eos
              ansible_become_method: enable
              ansible_become: true
        access_switches:
          hosts:
            pod1sw3:
              ansible_host: 172.16.1.4
              ansible_network_os: eos
              ansible_become_method: enable
              ansible_become: true
```
Each host will be defined under the lower groupings (routers, core_switches, and access_switches). We can store variables for our playbooks in these folders, including sensitive information like passwords. In this workshop, we will keep information like usernames and passwords under the group_vars/all folder in a file called "all.yml" If you were to have devices or device groups that did not use the same password, then you could create a file of the device name under the host_vars/{{ device_name }} folder. In this workshop all devices share a login. If you want to learn more about encrypting files with ansible use the ansible docs on [Ansible Vault](https://docs.ansible.com/ansible/latest/user_guide/vault.html)
```
---
# POD devices default login
ansible_user: ansible
ansible_password: ansible
```

## Section 3: Building Roles for the Access switch
We have several tasks to complete the deployment of our pod. We will be breaking each tasks down into its own play. This will be called a [Role](https://docs.ansible.com/ansible/latest/user_guide/playbooks_reuse_roles.html). This workshop Pod will be making use of the Cisco vIOS router and vEOS switch.

Lets create some folders to help structure where we will be placing data that will be used in our Ansible Plays.

Create new folders under the invnetory folder with the following structure:
```
inventory/
  group_vars/
    podx/
      pod1.yml
  host_vars/
    podxr1/
    podxsw1/
    podxsw2/
    podxsw3/
```
```
roles/
  access_switch/
  core_switch/
  routers/
```
In the roles/access_switch folder create the following structure:
```
add_access_interface/
  tasks/
  templates/
add_trunk_interface/
  tasks/
  templates/
add_vlan/
  meta/
  tasks/
  templates/
```
### Vlans and Access Ports
Create a new file under 'inventory/host_vars/podxsw3/' called 'vlans.yml'. In this file we will create a list of vlans that we need to create on our access switch and can reuse this same file for our core switches later on. Place the following text in your file:
```
---
configuration:
  vlans:
    vlan:
      - name: "USERS"
        vlan_id: "300"
        interface: eth3

      - name: "SERVERS"
        vlan_id: "350"
        interface: eth4

      - name: "GUEST"
        vlan_id: "400"
        interface: eth5

      - name: "NATIVE_VLAN"
        vlan_id: "666"
```

Create a new file under 'add_vlan/tasks/' called 'main.yml'. This task will create and name the vlans for the 3 user groups (Users, Servers, Guests) and a second task will assign an interface to each of the vlans created. This will utilize the ansible arista module to create the vlans and assign the interfaces for us. All we will need to provide are the variables and ansible will do the rest for us. This is one approach to pushing changes to a network device. 

In the main.yml file we will use the playbook tasks structure. This task will make use of the [Ansible Arista Vlan Module](https://docs.ansible.com/ansible/latest/collections/arista/eos/eos_vlan_module.html#ansible-collections-arista-eos-eos-vlan-module)

```
---
- name: Create vlan
  arista.eos.eos_vlan:
    vlan_id: "{{ item.vlan_id }}"
    name: "{{ item.name}}"
    state: present
  loop: "{{ configuration.vlans.vlan }}"

- name: Add interfaces to vlan
  arista.eos.eos_vlan:
    vlan_id: "{{ item.vlan_id }}"
    state: present
    interfaces: "{{ item.interface }}"
  loop: "{{ configuration.vlans.vlan }}"
  when: item.interface is defined
```

In a playbook a task is defined as the top item in a list and normally has a name that will be used to diplay to your command line the thing that is being done to whatever host Ansible is making changes to or gathering information from. Under the name will call the module that will be used for this task, followed by the modules required or optional parameters that will be changed or gathered on the managed node. 

Ansible allows us to loop through a list and provide those as variables to a module. In our case we created the vlans.yml file which is a list of configuration items for Switch3 in our pod. When we present a list to be looped over that is nested like ours we just need to tell the loop where to look for the list of items. We present variables to ansible modules by enclosing them in quotes ```""``` and double brackets ```{{}}```, in the loop statement we will tell the loop where to look for our list of things ```"{{ configuration.vlans.vlan }}"```. Now that ansible is looping through the lists we provided we need to tell it what items to pull out of the list. vlan_id, and name. When looping through a list Ansible calls the location "item" this replaces the location name we provided in the list "vlan". 
```"{{item.vlan_id}}"``` maps to our vlans.yml under configuration.vlans.vlan.vlan_id.

The module will loop through the 4 items on the list, and stop when the loop has completed.

Our next task will be to assign these vlans to their repesctive interfaces. We will make use of the loop again except this time we will use a "when" condition. Conditions are really handy, they allow us to create lists with only the item necessary, and if there is one missing it can just skip the item and continue on. In our vlan list we did not define a interface for the native vlan. Without the condition this play will fail when it encounters the native vlan becuase interface is required in the module and not present. With the conditional ```when: item.interface is defined``` this will only add an interface if "interface:" is defined in the list. 

### Trunk Ports
Create a new file under 'inventory/host_vars/podxsw3/' called 'trunk_interface.yml'. In this file we will create a list of trunk interfaces that we need to configure on our access switch.
Place the following text in your file:
```
---
configuration:
  interfaces:
    trunk:
      - name: Ethernet1
        description: "TRUNK TO POD1SW1"
        interface_mode: trunk
        native_vlan:
          members: "666"

      - name: Ethernet2
        description: "TRUNK TO POD1SW2"
        interface_mode: trunk
        native_vlan:
          members: "666"
```

Create a new file under 'add_trunk_interface/tasks/' called 'main.yml'. In this play we will configure our trunk interfaces.

```
- name: Configure layer2 trunk interfaces
  arista.eos.eos_l2_interfaces:
    config:
    - name: "{{ item.name }}"
      mode: "{{ item.interface_mode }}"
      trunk:
        native_vlan: "{{ item.native_vlan.members }}"
    state: replaced
  loop: "{{ configuration.interfaces.trunk }}"
```

Now we need to create a playbook for the above tasks. In your root folder create a file pb.setup.office.yml

```
- name: Configuring access switches
  hosts: access_switches
  gather_facts: false
  connection: network_cli

  roles:
    - { role: access_switch/add_vlan }
    - { role: access_switch/add_trunk_interface }
```
## Section 4: Building Roles for the Core Switches
### Tasks
2. (2) Core switches
    * Configure the following vlans:
      * Users - Vlan 300
      * Servers - Vlan 350
      * Guests - Vlan 400
      * Native Vlan - Vlan 666
    * Configure Layer 2 trunk ports to the access switch on ports Ethernet3 and between the Core switches on Ethernet2
    * Configure a Layer 2 port channel between both core switches on Ports Gi0/1 and Gi0/2
    * Configure SVIs for the following: 
      * Users - IP 155.x.1.0/26
      * Servers - IP 155.x.1.64/26
      * Guest - IP 155.x.1.128/26
    * Configure VRRP protocol for redundancy on above vlans
      * Use the first IP in the scope as your gateway
    * Configure Layer 3 interfaces as UPLINKS to the router
      * Core Switch 1 Port Ethernet0 with IP 10.x0.1.1/31
      * Core Switch 2 Port Ethernet0 with IP 10.x0.1.3/31
    * Configure OSPF to the above interfaces and advertise Loopback0
    * Configure Loopback0 interface to facilitate iBGP protocol peering
      * Core Switch 1 Loopback0 IP 10.x.1.2/32
      * Core Switch 2 Loopback0 IP 10.x.1.3/32
    * Use iBGP to advertise Users, Servers, and Guest subnets to the router
      * Use AS 6500x

Copy the vlans.yaml and trunk_interfaces.yaml files from your podxsw3 host_vars folder and place those into both the podxsw1 and podxsw2 folders. Remove the interfaces from the vlan.yml file and we will need to update the trunk_interfaces.yml file so that they both accuratly describe the connectivity of both switches.

Next lets describe our Layer3 interfaces, OSPF, and BGP, use the structure below. Place a l3_interfaces.yml file under switch1 and switch2 folders under your host_vars folder.

Examples are of Switch1, but both switches will need these files.
l3_interfaces.yml - under inventory/host_vars/podxsw1
```
---
configuration:
  interfaces:
    l3_interfaces:
      - name: vlan300
        description: "USER_SVI"
        ipv4: 155.1.1.2
        ipv4_mask: 255.255.255.192
        dhcp_helper: 10.0.1.1
        vrrp_group: 1
        vrrp_description: USER_VLAN
        vrrp_priority: 200
        vrrp_primary_ip: 155.1.1.1

      - name: vlan350
        description: "SERVER_SVI"
        ipv4: 155.1.1.66
        ipv4_mask: 255.255.255.192
        dhcp_helper: 10.0.1.1
        vrrp_group: 2
        vrrp_description: USER_VLAN
        vrrp_priority: 200
        vrrp_primary_ip: 155.1.1.65

      - name: vlan400
        description: "GUEST_SVI"
        ipv4: 155.1.1.130
        ipv4_mask: 255.255.255.192
        dhcp_helper: 10.0.1.1
        vrrp_group: 3
        vrrp_description: GUEST_VLAN
        vrrp_priority: 200
        vrrp_primary_ip: 155.1.1.129

      - name: Ethernet1
        description: "UPLINK POD1R1"
        ipv4: 10.10.1.1
        ipv4_mask: 255.255.255.254
        ospf:
          area: 0
          network: "point-to-point"

      - name: Loopback0
        description: "iBGP LOOPBACK"
        ipv4: 10.1.1.2
        ipv4_mask: 255.255.255.255
        ospf:
          area: 0
          network: "point-to-point"
```
 
ospf.yml - location of this file should be under inventory/host_vars/podxsw1
```
---
configuration:
  ospf:
    instance: 1
    router_id: 10.1.1.2
```

bgp.yml - location of this file should be under inventory/host_vars/podxsw1
```
---
configuration:
  bgp:
    ibgp:
      l_asn: 65001
      neighbors:
        - 10.1.1.1
        - 10.1.1.3
    address_family_ipv4:
      advertised_networks:
        155.1.1.0: {net_mask: 255.255.255.192 }
        155.1.1.64: {net_mask: 255.255.255.192 }
        155.1.1.128: {net_mask: 255.255.255.192 }
```

Lets take a look at creating configuration templates. This provides some flexability that using the ansible modules may not when pushing configurations. One of the things I like with templates if the ability to use the CLI commands which I am more famaliar with. It also provides the ability to incorporate several related tasks together under a single task. We will be using Jinja2 to build configuration templates for the core switches and the router. 

Lets start with the OSPF. 

add_ospf.j2 - location of this file should be under roles/core_switch/add_ospf/templates

{% raw %}
```
#jinja2: lstrip_blocks: "True (or False)", trim_blocks: "True (or False)"
{#- ---------------------------------------------------------------------------------- #}
{# configuration.ospf                                                                  #}
{# ---------------------------------------------------------------------------------- -#}
{% if configuration.ospf is defined %}
router ospf {{ configuration.ospf.instance }}
    router-id {{ configuration.ospf.router_id }}
    passive-interface Loopback 0
{% endif %}
```

Next the Layer3 interfaces. This template will make use of IF statements and loops to express the desired configuration to the nodes. We will include all of the interface cli commands required for the configure the interface. 

add_l3_interfaces.j2 - location of this file should be under roles/core_switch/add_l3_interfaces/templates
{% raw %}
```
#jinja2: lstrip_blocks: "True (or False)", trim_blocks: "True (or False)"

{% if configuration.interfaces.l3_interfaces is defined %}
  {% for l3_interface in configuration.interfaces.l3_interfaces %}
    {% if 'Loopback0' in l3_interface.name %}
interface {{ l3_interface.name }}
      {% if l3_interface.description is defined %}
 description {{ l3_interface.description}}
      {% endif %}
 ip address {{ l3_interface.ipv4 }} {{ l3_interface.ipv4_mask }}
      {% if l3_interface.ospf is defined %}
 ip ospf area {{ l3_interface.ospf.area }}
      {% endif %}
 no shut     
    {% elif 'Ethernet' in l3_interface.name and l3_interface.ospf is defined %}
interface {{ l3_interface.name }}
 no switchport
       {% if l3_interface.description is defined %}
 description {{ l3_interface.description }}
       {% endif %}
 ip address {{ l3_interface.ipv4 }} {{ l3_interface.ipv4_mask }}
 ip ospf network {{ l3_interface.ospf.network }}
 ip ospf area {{ l3_interface.ospf.area }}
 no shut   
    {% elif 'vlan' in l3_interface.name and l3_interface.vrrp_group is defined %}
interface {{ l3_interface.name }}
      {% if l3_interface.description is defined %}
 description {{ l3_interface.description}}
      {% endif %}
 ip address {{ l3_interface.ipv4 }} {{ l3_interface.ipv4_mask }}
 no shut
      {% if l3_interface.dhcp_helper is defined %}
 ip helper-address {{ l3_interface.dhcp_helper }}
      {% endif %}
 vrrp {{ l3_interface.vrrp_group }} ipv4 {{ l3_interface.vrrp_primary_ip }}
 vrrp {{ l3_interface.vrrp_group }} session description {{ l3_interface.vrrp_description }}
 vrrp {{ l3_interface.vrrp_group }} priority {{ l3_interface.vrrp_priority }}
    {% else %}
interface {{ l3_interface.name }}
      {% if l3_interface.description is defined %}
 description {{ l3_interface.description}}
      {% endif %}
 ip address {{ l3_interface.ipv4 }} {{ l3_interface.ipv4_mask }}
 no shut   
    {% endif %}
  {% endfor %}
{% endif %}
```
{% endraw %}

Now lets build the template for BGP. 

add_bgp.j2 - location of this file should be under roles/core_switch/add_bgp/templates
{% raw %}
```
#jinja2: lstrip_blocks: "True (or False)", trim_blocks: "True (or False)"

{% if configuration.bgp is defined %}
router bgp {{ configuration.bgp.ibgp.l_asn }}
  bgp router-id {{ configuration.ospf.router_id }}
  {% for ibgp_peers in configuration.bgp.ibgp.neighbors %}
  neighbor {{ ibgp_peers }} remote-as {{ configuration.bgp.ibgp.l_asn }}
  neighbor {{ ibgp_peers }} update-source Loopback0
  {% endfor%}
  {% if configuration.bgp.ebgp is defined %}
    {% for ebgp_peers,ebgp_peers_attr in configuration.bgp.ebgp.neighbors.items() %}
  neighbor {{ ebgp_peers }} remote-as {{ ebgp_peers_attr.r_asn }}
    {% endfor %}
  {% endif %}
  address-family ipv4
  {% for ibgp_peers in configuration.bgp.ibgp.neighbors %}
   neighbor {{ ibgp_peers }} activate
   neighbor {{ ibgp_peers }} next-hop-self
  {% endfor %}
  {% if configuration.bgp.ebgp.neighbors is defined %}
    {% for ebgp_peers,ebgp_peers_attr in configuration.bgp.ebgp.neighbors.items() %}
   neighbor {{ ebgp_peers }} activate
    {% endfor%}
  {% endif %}
  {% if configuration.bgp.address_family_ipv4.advertised_networks is defined %}
    {% for adv_nets,adv_nets_attr in configuration.bgp.address_family_ipv4.advertised_networks.items() %}
   network {{ adv_nets }} mask {{ adv_nets_attr.net_mask }} 
    {% endfor %}
    {% if configuration.bgp.address_family_ipv4.agg_network is defined %}
   aggregate-address {{ configuration.bgp.address_family_ipv4.agg_network }} {{ configuration.bgp.address_family_ipv4.agg_mask }} summary-only
    {% endif %}  
  {% endif %}
  exit-address-family
{% endif %}
```
{% endraw %}

We will need to also create a main.yml in the above folders where we just created the templates folder for our Jinja templates. 

We will use the eos_config module to push the lines generated from the template to each node. Besure to update the SRC with the file name of your template.

{%raw%}
```
- name: render a Jinja2 template onto an Arista switch
  arista.eos.eos_config:
    src: add_bgp.j2
    save_when: modified
```
{%endraw%}

Now that we have our templates and tasks created add on to the playbook we created previously pb.setup.office.yml.

```
- name: Configuring access switches
  hosts: access_switches
  gather_facts: false
  connection: network_cli

  roles:
    - { role: access_switch/add_vlan }
    - { role: access_switch/add_trunk_interface }
    
    
- name: Configuring core switches
  hosts: core_switches
  gather_facts: false
  connection: network_cli
  
  roles:
    - { role: core_switch/add_vlan }
    - { role: core_switch/add_trunk_interface }
    - { role: core_switch/add_l3_interfaces }
    - { role: core_switch/add_ospf }
    - { role: core_switch/add_bgp }
```

## Section 5: Building Roles for the Router
### Tasks
* Configure Layer 3 interfaces as DOWNLINKS to both core switches
  * Port Ethernet1 to Core Switch 1 with IP 10.x0.1.0/31
  * Port Ethernet2 to Core Switch 2 with IP 10.x0.1.2/31
* Configure Loopback0 interface to facilitate iBGP protocol
  * IP 10.x.1.1/32
* Configure OSPF to facilitate iBGP protocol
* Configure iBGP receive advertised Users, Servers, and Guest subnets from the core switches
  * Use AS 6500x
* Configure Port Gi0/0 with IP 24.24.x.2/24
  * The ISP ASN is 400 and the ISP IP address will be 24.24.x.1
  * Configure eBGP to advertise and Aggregate subnet from the Users, Servers, and Guest subnets 
  * Accept a default route from the ISP


We will use the same method to configure the router by expressing the intended configuration to the node with Jinja2 templates. 

New files will need to be created under the host_vars/podxr1 folder

bgp.yaml - location of this file should be under inventory/host_vars/podxr1
```
---
configuration:
  bgp:
    ibgp:
      l_asn: 65001
      neighbors:
        - 10.1.1.2
        - 10.1.1.3
    ebgp:
      neighbors: 
        24.24.1.1: {r_asn: 400}
    address_family_ipv4:
      agg_network: 155.1.1.0
      agg_mask: 255.255.255.0
      advertised_networks:
        155.1.1.0: {net_mask: 255.255.255.192 }
        155.1.1.64: {net_mask: 255.255.255.192 }
        155.1.1.128: {net_mask: 255.255.255.192 }
```

ospf.yaml - location of this file should be under inventory/host_vars/podxr1
```
---
configuration:
  ospf:
    instance: 1
    router_id: 10.1.1.1
```

l3_interfaces.yaml - location of this file should be under inventory/host_vars/podxr1
```
---
configuration:
  interfaces:
    l3_interfaces:
      - name: GigabitEthernet0/0
        description: "UPLINK TO INTERNET PROVIDER"
        ipv4: 24.24.1.2
        ipv4_mask: 255.255.255.0

      - name: GigabitEthernet0/1
        description: "DOWNLINK POD1SW1"
        ipv4: 10.10.1.0
        ipv4_mask: 255.255.255.254
        ospf:
          area: 0
          network: "point-to-point"

      - name: GigabitEthernet0/2
        description: "DOWNLINK POD1SW2"
        ipv4: 10.10.1.2
        ipv4_mask: 255.255.255.254
        ospf:
          area: 0
          network: "point-to-point"        

      - name: Loopback0
        description: "iBGP LOOPBACK"
        ipv4: 10.1.1.1
        ipv4_mask: 255.255.255.255
        ospf:
          area: 0
          network: "point-to-point"
```

Now that we have the variables defined create a new set of folders under roles/routers, add_bgp, add_l3_interfaces, and add_ospf.

The IOS module is slightly different than the Arista module. We will use the example below for each of the tasks.

main.yml
```
- name: configuring bgp on {{ inventory_hostname }}
  cisco.ios.ios_config:
    src: add_bgp.j2

- name: Saving the running config on {{ inventory_hostname }}
  ios_config:
    save_when: always
```

Our Jinja2 templates will be similar to the arista templates, but will vary slightly as some commands are different.

add_bgp.j2
{%raw%}
```
#jinja2: lstrip_blocks: "True (or False)", trim_blocks: "True (or False)"

{% if configuration.bgp is defined %}
router bgp {{ configuration.bgp.ibgp.l_asn }}
    bgp router-id {{ configuration.ospf.router_id }}
    {% for ibgp_peers in configuration.bgp.ibgp.neighbors %}
    neighbor {{ ibgp_peers }} remote-as {{ configuration.bgp.ibgp.l_asn }}
    neighbor {{ ibgp_peers }} update-source Loopback0
    {% endfor %}
    {% if configuration.bgp.ebgp is defined %}
        {% for ebgp_peers,ebgp_peers_attr in configuration.bgp.ebgp.neighbors.items() %}
    neighbor {{ ebgp_peers }} remote-as {{ ebgp_peers_attr.r_asn }}  
        {% endfor %}
    {% endif %}
    address-family ipv4
        {% for ibgp_peers in configuration.bgp.ibgp.neighbors %}
        neighbor {{ ibgp_peers }} activate
        neighbor {{ ibgp_peers }} next-hop-self
        {% endfor %}
        {% if configuration.bgp.ebgp.neighbors is defined %}
        {% for ebgp_peers,ebgp_peers_attr in configuration.bgp.ebgp.neighbors.items() %}    
        neighbor {{ ebgp_peers }} activate
        {% endfor %}
        {% endif %}
        {% if configuration.bgp.address_family_ipv4.agg_network is defined %}
        aggregate-address {{ configuration.bgp.address_family_ipv4.agg_network }} {{ configuration.bgp.address_family_ipv4.agg_mask }} summary-only
        {% endif %}
        {% if configuration.bgp.address_family_ipv4.advertised_networks is defined %}
            {% for adv_nets,adv_nets_attr in configuration.bgp.address_family_ipv4.advertised_networks.items() %}
        network {{ adv_nets }} mask {{ adv_nets_attr.net_mask }}
            {% endfor %}
        {% endif %}   
{% endif %}
```
{%endraw%}

add_l3_interfaces.j2
{%raw%}
```
#jinja2: lstrip_blocks: "True (or False)", trim_blocks: "True (or False)"

{% if configuration.interfaces.l3_interfaces is defined %}
{% for l3_interface in configuration.interfaces.l3_interfaces %}
{% if 'Loopback0' in l3_interface.name %}
interface {{ l3_interface.name }}
    {% if l3_interface.description is defined %}
    description {{ l3_interface.description}}
    {% endif %}
    ip address {{ l3_interface.ipv4 }} {{ l3_interface.ipv4_mask }}
    {% if l3_interface.ospf is defined %}
    ip ospf network point-to-point
    ip ospf 1 area 0
    {% endif %}
    no shut
{% elif 'GigabitEthernet' in l3_interface.name and l3_interface.ospf is defined  %}
interface {{ l3_interface.name }}
    {% if l3_interface.description is defined %}
    description {{ l3_interface.description}}
    {% endif %}
    ip address {{ l3_interface.ipv4 }} {{ l3_interface.ipv4_mask }}
    {% if l3_interface.ospf is defined %}
    ip ospf network point-to-point
    ip ospf 1 area 0
    {% endif %}
    no shut
{% else %}
interface {{ l3_interface.name }}
    {% if l3_interface.description is defined %}
    description {{ l3_interface.description}}
    {% endif %}
    ip address {{ l3_interface.ipv4 }} {{ l3_interface.ipv4_mask }}
    no shut
{% endif %}
{% endfor%}
{% endif %}
```
{%endraw%}

add_ospf.j2
{%raw%}
```
#jinja2: lstrip_blocks: "True (or False)", trim_blocks: "True (or False)"

{% if configuration.ospf is defined %}
router ospf {{ configuration.ospf.instance }}
    router-id {{ configuration.ospf.router_id }}
    passive-interface Loopback 0
{% endif %}
```
{%endraw%}

Now that we have our templates and tasks created add on to the playbook we created previously pb.setup.office.yml.

```
- name: Configuring access switches
  hosts: access_switches
  gather_facts: false
  connection: network_cli

  roles:
    - { role: access_switch/add_vlan }
    - { role: access_switch/add_trunk_interface }
    
    
- name: Configuring core switches
  hosts: core_switches
  gather_facts: false
  connection: network_cli
  
  roles:
    - { role: core_switch/add_vlan }
    - { role: core_switch/add_trunk_interface }
    - { role: core_switch/add_l3_interfaces }
    - { role: core_switch/add_ospf }
    - { role: core_switch/add_bgp }

- name: Configuring routers
  hosts: routers
  gather_facts: false
  connection: network_cli

  roles:
    - { role: routers/add_l3_interfaces }
    - { role: routers/add_ospf }
    - { role: routers/add_bgp }
```