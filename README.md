# RPC-Heat-VPN
Heat template for deploying VPN concentrator with strongswan



## Description
The Rackspace Private Cloud VPN Heat Solution, or RPC-Heat-VPN for short, is a Heat template designed to create and update a VPN concentrator using open sourced technology and the Salt configuration management engine.

The creation has two phases. The first phase is handled by Heat, laying down the necessary OpenStack resources for a fully functional VPN environment. Through Heat Software Deployments and Software Configs, the second phase is initiated. The second phase involves running salt states witch install strongswan and lay down the necessary configurations. A more detailed description of the solution will be provided below.

## Architecture
The diagram below best illustrates the architecture of this solution. A transient network is used so the VPN concentrator can route traffic to the tenant networks. VPN traffic is routed to the VPN concentrator through the neutron router by adding a static route. Indeed, multiple tenant networks are supported! Arbitrary tenant networks can be specified through heat parameters. Heat parameters are explained in the section below.

![](http://718016a9d23737f3d804-7671e86526a10735410d8ae5040e7d55.r41.cf1.rackcdn.com/VPN%20architecture%20diagram.png)


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

## Deployment - Under the Hood
Under the hood, strongswan is deployed and configured using the SaltStack configuration management engine. The RPC-Heat-VPN heat stack template leverages software deployment and software config resources to pull down the necessary salt states from a list of repos defined in the heat template. Salt configs and salt files containing sensitive information are also built using software config scripts. The software config scripts build these sensitive files using the parameters described above. 
Because this solution utilizes software configuration management, additional functionality can be easily added by pulling in new salt states. Thus, true DevOps principles are achieved.

###Logging
All software config scripts are appropriately logged on the VPN concentrator in the /var/log/heat-deployments directory. The log files are named according to their corresponding heat resource. For example, the config-build-pillar software config resource will log to /var/log/heat-deployments/config-build-pillar.log, and so on.

### Troubleshooting
Remote access to the VPN concentrator can be done by using the key given during stack creation. For example:

```shell
ssh -i <key> ec2-user@<floating-ip-of-concentrator>
```

You can double check the /etc/ipsec.conf and /etc/ipsec.secrets file for an rendoring errors. Also, you can checkout the ipsec logs by grepping for 'charon' in the syslog file. 

## Quick Start Guide
### Creating a stack
Creating a stack is simple. Just make sure the networks and router that will be utilizing the VPN are in place. Also, make sure you have access to the neutron API through the neutron client:

1. First, clone this repository. 

2. Access the openstack dashboard using your credentials.

3. Navigate to the Orchestraion->Stacks tab.

5. Click on Launch Stack

6. Make sure Template Source is from File, then upload the vpn-stack.yaml file.

7. Enter your parameters. Hit Launch.

8. When the stack is finished, copy-paste the neutron port-update command in the Outputs section of the stack. 

Image to be inserted
### Updating a stack
Updating a stack is an expiermental feature at the moment, so please be careful. Only adding a VPN user and a VPN left network have been tested.
To update a stack: 

1. Navigate to where your stacks are listed on the Openstack dashboard, click on the VPN stack you want to update. 

2. Upload the SAME vpn-stack.yaml template used on creation. 

3. Enter the SAME parameters as when you created the stack, with the exception of Left Networks and/or Vpn Users. 

4. To add a Left Network or VPN user, simply list them in comma delimited form in their appropriate parameter fields. 

5. Hit "Update". 

WARNING: Be careful with adding users and/or left networks. Adding duplicates will currently break your VPN. 
