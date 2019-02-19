# Opendaylight VXLAN demo

## Overview

Purpose of this usecase is to show possibility of engagement Cloudify as an orchestrator responsible for SDN service provisioning.

In this example Opendaylight SDN controller is used to provision VXLAN tunnel between 2 OpenVSwitches. 
Service creation is automated using Cloudify - it communicates with Opendaylight northbound RESTCONF API to invoke topology-related operations, service creation and deletion etc.

To show described above scenario - virtual infrastructure on Openstack is used.
It is also created by Cloudify and consist of:
* client VM
* server VM runs simple HTTP server
* VM with Opendaylight controller
* 2 VMs with OpenVSwitches
* appropriate networks, subnets, security groups etc.

Whole setup is presented below: 
 
![opendaylight_vxlan](https://user-images.githubusercontent.com/20417307/34303949-f6770354-e737-11e7-9301-dfc2bf0c90e0.png)

Solution consists of 2 blueprints:
* ***opendaylight_infrastructure_blueprint*** - creates all VMs, networks, other openstack resources, installs required software (ovs, odl) and push initial configs,
* ***opendaylight_service_configuration_blueprint*** - creates service (VXLAN tunnel) by calling ODL RESTCONF northbound API

Additional files:
* ***imports/cloud_config.yaml*** - file that holds CloudConfigs for all VMs.
* ***restconf/\*.yaml*** - YAML HTTP RESTCONF request templates used to configure ODL 

In this example ***Opendaylight v.0.6.2-Carbon*** is used. 

## Demo

### Prerequisites

* ***Cloudify Manager***

**This demo has been designed to be run in Cloudify LABS (labs.cloudify.co)**. 
However, it is also possible to run it on standalone Cloudify Manager.

* ***Plugins***

    Two plugins are required:
    * **cloudify-openstack-plugin v.2.14.7**
    
    ```
        cfy plugins upload https://github.com/cloudify-cosmo/cloudify-openstack-plugin/releases/download/2.14.7/cloudify_openstack_plugin-2.14.7-py27-none-linux_x86_64-centos-Core.wgn -y https://github.com/cloudify-cosmo/cloudify-openstack-plugin/releases/download/2.14.7/plugin.yaml 
    ```
    
    * **cloudify-utilities-plugin v.1.11.2**
    
    ```
        cfy plugins upload https://github.com/cloudify-incubator/cloudify-utilities-plugin/releases/download/1.11.2/cloudify_utilities_plugin-1.11.2-py27-none-linux_x86_64-centos-Core.wgn -y https://github.com/cloudify-incubator/cloudify-utilities-plugin/releases/download/1.11.2/plugin.yaml
    ```
* ***Secrets*** 

    * In ***Cloudify Labs*** all the of secrets variables are already added
    * To run this example on standalona Cloudify Manger you need to define these secrets:
        * ***agent_key_public*** - public key which will be used to connect to each VM using SSH
        * ***keystone_username*** - username used for authentication to Openstack Keystone API
        * ***keystone_password*** - password used for authentication to Openstack Keystone API
        * ***keystone_tenant_name*** - Openstack tenant name
        * ***keystone_region*** - Openstack region
        * ***keystone_url*** - URL to Openstack Keystone API
        * ***external_network_name*** - Name or ID of Openstack external (public) network (which has Internet connectivity)
        * ***private_network_name*** - Name or ID of Openstack provate network which will be use to interconnect VMs 
        * ***centos_core_image*** - ID of Openstack image with CentOS (for tests *CentOS 7* was used)
        * ***small_image_flavor*** - ID of Openstack flavor with 1 vCPU, 2 GB RAM and 20 GB disk 
        * ***medium_image_flavor*** - ID of Openstack flavor with 2 vCPU, 4 GB RAM and 40 GB disk 

    Secrets can be added using CLI command:
    
    ```
       cfy secrets create <secret_name> -s <secret_value> 
    ```

### Execution

1) Install ***opendaylight_infrastructure_blueprint***:
    ```
        cfy install opendaylight_infrastructure_blueprint.yaml -d odl_infrastructure
    ```
    
    This blueprint creates setup described in Overview section.

2) **IMPORTANT !**

    Wait about 10 mins after deployment installation is succeeded. 
    There are a lot of configuration activities being still in progress (even when deployment instalation has ended).
    This is because of Cloud Init usage - Cloudify doesn’t wait till end of the execution so even after deployment installation some configuration activities are not done.
    
    ***This step should be finally removed when issue#6 will be closed***

3) Check outputs for ***odl_infrastructure*** deployment. 
   Get ***service_configuration_inputs*** output and use its entries as inputs for ***opendaylight_service_configuration_blueprint*** deployment creation in ***step 4***.
     
    ```
        cfy deployments outputs odl_infrastructure
    ```

4) Run Cloudify configuration blueprint.
    ```
        cfy install opendaylight_service_configuration_blueprint.yaml -b odl_service -i odl_ip="<value>" -i odl_port="<value>" -i a_endpoint="<value>" -i z_endpoint="<value>"
    ```

5) **IMPORTANT !**

    Take ***server_config*** output of ***odl_infrastructure*** deployment.
    Execute CLI commands specified in this outputs on ***server VM*** using SSH.
    Do the same with ***client VM*** and ***client_config*** outputs. 
    These commands sets ARP table entry and IP route necessary to reach server from client and client from server.
    SSH command should be:
    
    ```
        ssh centos@<floating_ip_of_vm> -i <path_to_private_key_for_odl_key_pair>
    ```

    Floating IPs you can find in ***access*** output of ***odl_infrastructure*** deployment.
    
    ***This step should be finally removed when issue#7 will be closed***

6) Check service configuration (you can also use these commands to check behaviour before and after installation of service blueprint):

    * OVS A:
    ```
        ovs-vsctl show 
        ovs-ofctl show A
        ovs-ofctl dump-flows A
    ```

    * OVS Z:
    ```
        ovs-vsctl show 
        ovs-ofctl show Z
        ovs-ofctl dump-flows Z
    ```

    * Client - commands defined in ***client_check_commands*** infrastructure deployment outputs. Should be:
    ```
        ping <server IP from z_edn_network> 
        curl http://<server IP from z_edn_network>:8000
    ```

    IP address you can also get from infrastructure deployment outputs: ***topology > z_end_network > server***

7) Uninstall blueprints

    ```
        cfy uninstall odl_service
        cfy uninstall odl_infrastructure
    ```

## Troubleshooting

* Check cloud-init log:

```
    less /var/log/cloud-init-output.log
```

* Check if ODL is running (only one java process should be run):

```
    ps aux | grep java
```

* Run ODL manually:

```
    /home/centos/distribution-karaf-0.6.2-Carbon/bin/start
```

* Check if server is running (python -m SimpleHTTPServer process should be present):

```
    ps aux | grep python
```