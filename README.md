# Cumulus Putting It All Together demo

This demo will install Cumulus Linux (VX) in a 2-pod topology that was made with the [topology converter](https://github.com/CumulusNetworks/topology_converter). This demo puts several technologies together to demo a complete deployment:

* EVPN L2VNIs
* EVPN L3VNIs
  * IPv4 Type-5
  * IPv6 Type-5
* Cumulus NetQ
* VRF setup on Ubuntu 16.04 with Ifupdown2
* Netbox IPAM & automated provisioning of the Ansible inventory
* Grafana for graphs from the NetQ interface statistics
* CI/CD with Gitlab
* Custom Ansible Jinja templates to deploy the above


![cl-piat topology](diagrams/cl-piat-topology.png)


Table of Contents
=================
* [Prerequisites](#prerequisites)
* [Using Libvirt KVM](#using-libvirtkvm)
* [Using Virtualbox](#using-virtualbox)
* [Using Cumulus in the Cloud](#using-cumulus-in-the-cloud)
* [Demo explanation](#demo-explanation)
  * [Tenant setup](#tenant-setup)
  * [Automation](#automation-and-orchestration)
* [Running the Demo](#running-the-demo)
  * [EVPN](#EVPN)
  * [NetQ](#NetQ)
  * [Grafana](#Grafana)
  * [Netbox](#Netbox)
  * [CI/CD](#CI/CD)
* [Troubleshooting + FAQ](#troubleshooting--faq)


Prerequisites
------------------------
* Running this simulation roughly 23G of Ram (by default only pod1, superspines and edge are started. Full topology will use ~35G)
* Internet connectivity is required from the hypervisor. Multiple packages are installed on both the switches and servers when the lab is created.
* Download this repository locally with `git clone https://github.com/CumulusNetworks/cl-piat.git` 
* Download the NetQ Telemetry Server from https://cumulusnetworks.com/downloads/#product=NetQ%20Virtual&hypervisor=Vagrant. You need to be logged into the site to access this.  Choose NetQ 1.3.
* Setup your hypervisor according to the instructions with [Vagrant, Libvirt and KVM](https://docs.cumulusnetworks.com/display/VX/Vagrant+and+Libvirt+with+KVM+or+QEMU)

Using Libvirt+KVM
------------------------
* Install the Vagrant mutate plugin with 
`vagrant plugin install vagrant-mutate`
* Convert the existing NetQ telemetry server box image to a libvirt compatible version.   
`vagrant mutate cumulus-netq-telemetry-server-amd64-1.3.0-vagrant.box libvirt`
* Rename the new Vagrant box image by changing the Vagrant directory name.  
`mv $HOME/.vagrant.d/boxes/cumulus-netq-telemetry-server-amd64-1.3.0-vagrant/ $HOME/.vagrant.d/boxes/cumulus-VAGRANTSLASH-ts`
* Start the topology with the script `./evpn-symmetric-edge.sh`. This will run the vagrant commands to start the topology. By default only pod1, superspines and edge devices are started. This should provide sufficient devices for most demos, but the topology would give the possibility to run the demo on a larger environment. 

Next, when fully booted:
* Enter the environment by running `vagrant ssh oob-mgmt-server` from the vx-topology directory. On the oob-server, the same git repository is cloned that hold the necessary Ansible playbooks. Start provisioning the environment by running `./provision.sh` from the `cl-piat/automation` directory. This will run an ansible playbook that provisions the network devicees and servers. Running the full playbook when the topology isn't provisioned yet, can take several minutes, because several software packages have to be installed (be patient).

Using Virtualbox
------------------------
The topology can be generated for Virtualbox, but given the requirements it will not run on on typical desktops/laptops. Recommentation is to use the demo with the Libvirt/KVM. If Virtualbox is necessary, create an issue and we can create a Virtualbox Vagrantfile.  


Using Cumulus in the Cloud
------------------------
Request a "Blank Workbench" on [Cumulus in the Cloud](https://cumulusnetworks.com/try-for-free/). When you receive notice that it is provisioned, connect to the *oob-mgmt-server*

Once connected run  
`git clone https://github.com/CumulusNetworks/cl-piat.git`

Because the CITC workbench is a subset of the above topology (2 spines, 4 leafs, 4 servers), we have to change the hosts file to provision less hosts. There is a specific hosts file provided in the `demo` directory. 

Next  
* `cd cl-piat/automation`  
* `mv ../demo/citc/hosts .`
* `ansible network -m shell -a 'net add vrf mgmt; net commit'`
* `echo "127.0.0.1 netq-ts" | sudo tee --append /etc/hosts`
* `./provision.sh`

Demo explanation
------------------------
This section explains how the demo has been built. The "running demo" section shows some examples that can be done during a live demo.

### Tenant setup
The environment is built with three tenants that are routed in the EVPN L3 overlay. Each leaf is provisioned with three VRFs/L3VNIs. Each VRFs has two IPv4 and IPv6 SVIs that could be different services on a host. These SVIs are tagged in 6 vlans to each server. The servers are setup with a new kernel, VRF support and VRF tools. This is done for two reasons, 1. Show the development of Linux tools by Cumulus and how it is applicable beyond just switches, 2. makeing sure that each "service" has it's own routing table and can be configured with a default route (statically for IPv4, through RAs with IPv6).

![Tenants](diagrams/cl-piat-tenants.png)

This setup means that there is no local bridging between the "apps" on the servers. E.g traffic from app1 to app2 on the same host will be routed on the TOR. There is no traffic possible between "apps" in different VRFs unless route-leaking would be configured on either rtr01/rtr02. 

### Automation and Orchestration

This environment is fully provisioned with Ansible in combination with Jinja2 templates. Currently the playbook is separated in multiple roles for each type of component, such as leaf and spine switches. The servers and provisioning of management tools is also done with separate roles. This creates an easy way of setting up the base for the demo / PoC environment.  

![Orchestration](diagrams/cl-piat-orchestration.png)

In most demos we have shown how orchestration of the configuration can be done through editting the Ansible variable files. While this might work in some environments, this wouldn't be always the case. In an environment with infrastructure data in multiple places, you would like to have a single source of truth. In this case Netbox (opensource IPAM, developed bij DigitalOcean) was used to store the variables needed to configure the infrastructure. In this demo the Netbox is the source of truth and orchestration is done as shown in the above diagram. The script "netbox.py" (Netbox importer), uses the Netbox api to generate a json file that contains the variable file for Ansible. Ansible uses the variables to generate the configuration files based on Jinja templates.

Running the Demo
------------------------
When the demo is fully provisioned, there are several topics that can be shown during the demo depending on the audience.

### EVPN
The spine / leaf topology has been deployed with a symmetrical EVPN setup with the aforementioned tenant setup.


### NetQ
### Grafana
During deployment Grafana is installed as a container on the netq-ts server. To make the gui accessable, we need to setup a remote ssh tunnel to a host that is publicly accessible: `screen ssh -NR 3001:localhost:3000 <user>@<host>`. Exit the screen with ctcl+A,D. Unfortunately the current Ansible module for Grafana has a bug that prevents the dashboards to be automatically provisioned (https://github.com/ansible/ansible/issues/39663). When the interface is accessible (default user/pass: admin / CumulusLinux!), there are two dashboards available that can be imported into Grafana in the roles/telemetry/files directory. Import them both to have access to them.

The IP-fabric dashboard will display all interfaces on a specified switch:
![IP-Fabric](images/ip-fabric.png)

The detailed dashboard allows for a more granular selection of interfaces on multiple switches. As shown in the screenshot it will show traffic / errors of the interfaces swp1,2,49,50 of switches leaf01 and leaf02. This allows for showing traffic from up / downlink for specific servers.
![Detailed](images/detailed.png)

By default you will see only low traffic volumes. This repository also has an Ansible playbook that will use iperf to setup a full mesh of streams in all VRFs to show the traffic in the Grafana graphs. Use `ansible-playbook traffic.yaml -l servers` to start the traffic flows for 5 minutes.


### Netbox
### CI/CD


Troubleshooting + FAQ
-------
* The `Vagrantfile` expects the telemetry server to be named `cumulus/ts`. If you get the following error:

```
The box 'cumulus/ts' could not be found or could not be accessed in the remote catalog. If this is a private box on HashiCorp's Atlas, please verify you're logged in via `vagrant login`. Also, please double-check the name. The expanded URL and error message are shown below:

URL: ["https://atlas.hashicorp.com/cumulus/ts"]
Error: The requested URL returned error: 404 Not Found
```

Please ensure you have the telemetry server downloaded and installed in Vagrant. Use `vagrant box list` to see the current Vagrant box images you have installed.
* `vagrant ssh` fails to network devices - This is expected, as each network device connects through the `oob-mgmt-server`. Use `vagrant ssh oob-mgmt-server` then ssh to the specific network device.
* If you log into a switch and are prompted for the password for the `vagrant` user, issue the command `su - cumulus` to change to the cumulus user on the oob-mgmt-server
