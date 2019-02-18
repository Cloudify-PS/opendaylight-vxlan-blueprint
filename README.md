# Opendaylight VXLAN demo

## Overview

![opendaylight_vxlan](https://user-images.githubusercontent.com/20417307/34303949-f6770354-e737-11e7-9301-dfc2bf0c90e0.png)

Solution consists of 2 blueprints:
* ***opendaylight_infrastructure_blueprint*** - creates all VMs, networks, other openstack resources, installs required software (ovs, odl) and push initial configs,
* ***opendaylight_service_configuration_blueprint*** - creates vxlan by calling ODL RESTCONF using cloudify-rest-plugin (to finish)

Additional files:
* ***imports/cloud_config.yaml*** - file that holds CloudConfigs for all VMs.
* ***restconf/\*.yaml*** - YAML HTTP RESTCONF request templates used to configure ODL: 

## Demo

### Prerequisites
In order to run any of prepared blueprints in Cloudify Labs, some prerequisites must be fulfilled:

* ***Secrets store*** 

    Most of secrets variables are already added, to run first blueprint one have to be added:
    ***odl_key_public*** - After creation of keypair (locally), content of public key have to be added to this secret in order to be able to get ssh access to virtual machines with private key (created together with public one)

    ```
       cfy secrets create odl_key_public -s <content_of_key> 
    ```

* ***Wagons***

    At least ***cloudify-rest-plugin*** wagon have to be added to Cloudify.

    ```
        wagon create -f -t tar.gz -a <path to cloudify-rest-plugin repo>
        cfy plugin upload <path to wagon>
    ```

### Execution

With prerequisites fulfilled, prepared blueprints can be properly deployed.
Demo should be executed by following below steps:

1) Install ***opendaylight_infrastructure_blueprint***:
```
    cfy install opendaylight_infrastructure_blueprint.yaml -d odl_infrastructure
```

This blueprint creates setup described in Overview section.

2) Wait a while after deployment successful installation. 
There are a lot of configuration going on different VMs. Those are pushed with *cloudify.nodes.CloudInit.CloudConfig* and Cloudify doesn’t wait till end of the execution so even after deployment installation some configuration are not pushed. It should take around 10 minutes.

3) Check outputs for *odl_infrastructure* deployment. Get *service_configuration_inputs* from outputs and use it as inputs for ODL configuration deployment.
 
```
    cfy deployments outputs odl_infrastructure
```

4) Run Cloudify configuration blueprint.
```
    cfy install opendaylight_service_configuration_blueprint.yaml -b odl_service -i inputs.yaml
```

5) **IMPORTANT !**

    Before next step please take *server_config* CLI commands from outputs obtained in previous server and execute it on server VM using SSH.
    Please do the same with client VM and *client_config* outputs. 
    These commands sets ARP table entry and IP route necessary to reach server from client and client from server.
    SSH command should be:
    
    ```
        ssh centos@<floating_ip_of_vm> -i <path_to_private_key_for_odl_key_pair>
    ```

    Floating IPs you can find in *access* output

6) Check service configuration (you can use them also before and after provisioning to check if there are no configuration on OvSes):

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

    * Client:
    ```
        ping <IP from z_edn_network of server> 
        curl http://<IP from z_edn_network of server>:8000
    ```

    IP address you can get from infrastructure deployment outputs: *topology > z_end_network > server*

7) Uninstall blueprints

```
    cfy uninstall odl_service -p ignore_failure=True
    cfy uninstall odl_infrastructure -p ignore_failure=True   
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