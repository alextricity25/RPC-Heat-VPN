# RPC-Heat-VPN
Heat template for deploying VPN concentrator with strongswan



## Description
The Rackspace Private Cloud VPN Heat Solution, or RPC-Heat-VPN for short, is a Heat template designed to create and update a VPN concentrator using strongswan and the Salt configuration management engine.

The creation has two phases. The first phase is handled by Heat, laying down the necessary OpenStack resources for a fully functional VPN environment. Through Heat Software Deployments and Software Configs, the second phase is initiated. The second phase involves running salt states and python scripts witch install strongswan and lay down the necessary configurations. A more detailed description of the solution will be provided below.

## Quick Start Guide
### Creating a stack
Creating a stack is simple. Just make sure the networks and router that will be utilizing the VPN are in place. Also, make sure you have access to the neutron API through the neutron client:

* First, clone this repository. 

* Access the openstack dashboard using your credentials.

* Navigate to the Orchestraion->Stacks tab.

* Click on Launch Stack

* Make sure Template Source is from File, then upload the vpn-stack.yaml file.

* Enter your parameters. Hit Launch.

* When the stack is finished, copy the neutron port-update command in the Outputs section of the stack and run in on your neutron client with proper credentials. See image below for an example.

![](http://718016a9d23737f3d804-7671e86526a10735410d8ae5040e7d55.r41.cf1.rackcdn.com/Neutron_port_command.png)

* Record the usernames and passwords from the heat outputs in a secure location. They will be cleared on the next stack update. Alternatively, they can be cleared by passing the Users parameter field "clear passwords". See below section on clearing sensitive information.

### Updating a stack
Updating a stack is the the best part of this solution! Imagine if the user wanted to add a VPN user or a network for split tunneling. How can they do so without having dive into strongswan configurations? Stack updates are the answer! Using stack updates, the cloud administrator can add/delete a VPN user and network, update a password, and clear the sensitive information from the heat output section on the dashboard. There is no need for the admin to log into the VPN concentrator and dig through strongswan configurations! Detailed instructions on how to do these things are provided below. 

####Adding a user/network

* Navigate to where your stacks are listed on the Openstack dashboard, find the VPN stack you want to update, then select the "Change Stack Template" option.

![](http://718016a9d23737f3d804-7671e86526a10735410d8ae5040e7d55.r41.cf1.rackcdn.com/updating_stack.png)

* Upload the SAME vpn-stack.yaml template used on creation. 

* Enter the parameters that are listed. For the paramters you want to keep the same, simply enter the same value as before. However, keep in mind that updating the openstack resources hasn't been tested. Stick to only changing the Left Networks or Users field.

* To add a Left Network or VPN user, simply list them in comma delimited form in their appropriate parameter fields. For example, to add the network 10.10.10.0/24 and the user "testuser":

![](http://718016a9d23737f3d804-7671e86526a10735410d8ae5040e7d55.r41.cf1.rackcdn.com/Adding_entries.png)

If only a user needs to be added, leave the Left Networks field blank as it's an optional field. 

* Hit "Update". 

* Make sure to re-run the neutron port-update command that will appear in the heat outputs section if new networks were added.

The newly created users, along with their passwords, will appear in the outputs section. For example:

![](http://718016a9d23737f3d804-7671e86526a10735410d8ae5040e7d55.r41.cf1.rackcdn.com/heat_output.png)

####Deleting a user
Deleting a user is very simple! 
NOTE: Deleting a user will clear sensitive information from the heat outputs, such as usernames and passwords. 
To delete a user or a network: 

* Perform the same actions as discribed above for updating a stack. 

* In the 'Users' field, prefix the list of users or networks you want to delete with 'delete:'. For example: 

![](http://718016a9d23737f3d804-7671e86526a10735410d8ae5040e7d55.r41.cf1.rackcdn.com/delete.png)

* Click on "Update". 

Deleting users or networks can also be done in bulk by entering them as a comma delimited list. 

####Updating a users password
To update a users password, prefix the list of users that need the update with 'updatepw:' in the Users parameter field. For example:

![](http://718016a9d23737f3d804-7671e86526a10735410d8ae5040e7d55.r41.cf1.rackcdn.com/update_passwords.png)

The user will recieve a new randomly generated password, and will be displayed in the heat output section. 


####Clearing the sensitive information
To clear the sensitive information from the heat outputs, type 'clear passwords' in the Users parameter field on a stack update:

![](http://718016a9d23737f3d804-7671e86526a10735410d8ae5040e7d55.r41.cf1.rackcdn.com/clear_passwords.png)

The sensitive information will then be removed from the heat outputs on the dashboard.

## Heat Parameters
Heat parameters are user defined values that specify how the stack will be created and the VPN concentrator built. In this case, the following parameters are avialable to the user: 

* Image - The image used for the VPN concentrator. The image must have heat software config elements built into it. This deployment has only been tested on Ubuntu 14.04. An image can be found at http://ab031d5abac8641e820c-98e3b8a8801f7f6b990cf4f6480303c9.r33.cf1.rackcdn.com/ubuntu-trusty-software-config.qcow2
* External Network UUID - This is the neutron physical provider network. The floating IP for the VPN concentrator is grabbed from here. 
* Keyname - The key used to ssh into the VPN concentrator
* Neutron Router UUID - This is a router that should already be in place, with some tenant networks already attached. See image above. 

* VPN Group Name Prefix - This is the group name prefix used to connect to the VPN concentrator. It is appened with a random string to prevent duplicates. Default is RAXVPN-group.
* Left Networks - The tenant networks available from the VPN. See diagram above.
*VPN users - A comma delimited list of users on the VPN. Passwords are randomly generated and displayed as heat stack outputs. 
* DHCP pool cidr - This is the DHCP pool cidr for the VPN concentrator. Defaults to 192.168.238.0/24
* Transient Network - The network used by the VPN concentrator to route traffic to the tenant networks. Default is 172.29.255.0/24.

## Architecture
The diagram below best illustrates the architecture of this solution. A transient network is used so the VPN concentrator can route traffic to the tenant networks. VPN traffic is routed to the VPN concentrator through the neutron router by adding a static route. Indeed, multiple tenant networks are supported! Arbitrary tenant networks can be specified through heat parameters. Heat parameters are explained in the section below.

![](http://718016a9d23737f3d804-7671e86526a10735410d8ae5040e7d55.r41.cf1.rackcdn.com/VPN%20architecture%20diagram.png)


## Deployment - Under the Hood
Under the hood, strongswan is deployed and configured using the SaltStack configuration management engine. The RPC-Heat-VPN heat stack template leverages software deployment and software config resources to pull down the necessary salt states from a list of repos defined in the heat template. Salt files containing sensitive information (pillars) are built by a python script called by the "config-build-pillar" software config resource. Data from heat is passed to the script via arguments. 

Each time a stack is updated, the python script re-runs, re-building the salt pillar files with the newly added/deleted data. Salt then uses that information to configure strongswan accordingly. 

Because this solution utilizes software configuration management, additional functionality can be easily added by pulling in new salt states. Thus, true DevOps principles are achieved.

###Logs
All software config scripts are appropriately logged on the VPN concentrator in the /var/log/heat-deployments directory. The log files are named according to their corresponding heat resource. For example, the config-build-pillar software config resource will log to /var/log/heat-deployments/config-build-pillar.log, and so on.

### Troubleshooting
Remote access to the VPN concentrator can be done by using the key given during stack creation. For example:

```shell
ssh -i <key> ec2-user@<floating-ip-of-concentrator>
```

You can double check the /etc/ipsec.conf and /etc/ipsec.secrets file for an rendoring errors. Also, you can checkout the ipsec logs by grepping for 'charon' in the syslog file. 


