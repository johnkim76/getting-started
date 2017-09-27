Getting Started Guide - Early Access
====================================

## Introduction
```What is service catalog and what are brokers in your cloud service infrastructure?```

* Service Catalog is the component which finds and presents the users the list of services which the user has access to.  It also gives the user the ability to provision new instances of those services and provide a way to to bind the provisioned services to existing applications.

* The Brokers are the components that manages a set of capabilities in the cloud infrastructure, and provides the service catalog with the list of services, via implementing the [Open Service Broker API](https://github.com/openservicebrokerapi/servicebroker)

```What Amazon Services are exposed to Service Catalog?```

Various Amazon Web Services are exposed in the service catalog via the [Ansible Playbook Bundle (APB)](https://github.com/ansibleplaybookbundle/ansible-playbook-bundle) implementations of those services.

## Prerequisite
The following pre-reqs must be satisfied
  * OpenShift Container Platform (OCP) v3.7
  * Service Catalog Enabled
  * Ansible Service Broker configured with a valid docker registry

## Environment Setup
The following describe the steps to setup the various OpenShift Environments

### CatASB
[CatASB](https://github.com/fusor/catasb) is a collection of playbooks to create an OpenShift environment with a [Service Catalog](https://github.com/kubernetes-incubator/service-catalog) & [Ansible Service Broker](https://github.com/openshift/ansible-service-broker) in a local or EC-2 environment. CatASB will create an OpenShift environment which will allow the user to provision and deploy the APBs easily.

In order to use catasb, clone the following git repo
```bash
git clone https://github.com/fusor/catasb.git
cd catasb
```

Copy the `my_vars.yml.example` to `my_vars.yml`, and edit the file. ```Review``` the varialbes, and uncomment those that you wish to override. 
```bash
cp config/my_vars.yml.example config/my_vars.yml
vi config/my_vars.yml
```

Below are some of the variables that you may want to override:
```bash
dockerhub_user_name: <username>
dockerhub_user_password: <password>
dockerhub_org: ansibleplaybookbundle

deploy_origin: true
aws_custom_prefix: origintest1

broker_enable_basic_auth: false
broker_bootstrap_refresh_interval: 24h

origin_image_tag: latest
openshift_client_version: latest
broker_tag: canary
broker_ssl: false
``` 

#### Local
This will effectively do an `"oc cluster up"`, and install/configure the service catalog with the broker.

Navigate into the local/<OS> folder
```bash
cd local/linux  # for Linux OS

or

cd local/mac    # for MacOS
```
```Note```: if you are running in the MacOS environment, review and edit the `config/mac_vars.yml`

Run the setup script
```bash
./run_setup_local.sh  # for Linux OS

or

./run_mac_local.sh    # for MacOS
```
The script will output the details of the OpenShift Cluster

```Note```: When visiting the cluster URL (e.g. https://172.17.0.1:8443/console/), you may get an issue with not being able connect.  Check your firewall rules to make sure all of the OpenShift Ports are permitted. [Click here to see the list of ports](https://docs.openshift.com/container-platform/latest/install_config/install/prerequisites.html#required-ports)

#### Single-Node 
This environment uses `"oc cluster up"` in a single EC-2 instance, and will install the openshift components from RPMs.

Navigate into the ec2/minimal folder
```bash
cd ec2/minimal
```

Setup the AWS network, and the EC-2 instance:
```bash
./run_create_infrastructure.sh
``` 
The script will output the details of the AWS environment

Next, install and configure OpenShift, service catalog, and the broker
```bash
./run_setup_environment.sh
```
To terminate the EC-2 instance and to remove/clean-up the AWS network, run the following
```bash
./terminate_instances.sh
```

All of the scripts above will output the details of the OpenShift Cluster.  However, if you wish to review those details at any time, you can run the following:
```bash
./display_information.sh
```

#### Multi-node
This environment installs OpenShift with the service catalog and the broker via openshift-ansible in multiple EC-2 instances (1 master with 2 nodes)

Navigate into the ec2/multi_node folder
```bash
cd ec2/multi_node
```

Setup the AWS network, and the EC-2 instances:
```bash
./run_create_infrastructure.sh
``` 
The script will output the details of the AWS environment

Next, install and configure OpenShift, service catalog, and the broker via the OpenShift-Ansible
```bash
./run_openshift_ansible.sh
```

Create a admin and a develop user
``` bash
./run_post_aos_install.sh
```

To terminate the EC-2 instances, run the following: 
```bash
./terminate_instances.sh
```
Note: The above script ONLY removes the EC-2 instances.  It does NOT remove the AWS Network (i.e. VPC, subnets, gateways, r53 DNS entries, etc.).  

All of the scripts above will output the details of the OpenShift Cluster.  However, if you wish to review those details at any time, you can run the following:
```bash
./display_information.sh
```

### Minishift
```TBD - TODO```

## APB Guide

[Ansible Playbook Bundles (APBs)](https://github.com/ansibleplaybookbundle/ansible-playbook-bundle) defines the application definitions for the Amazon Web Services, and implemented so that the brokers can easily manage and provide the services to the services catalog.  

### APB Prerequisite
To use a new APB in an existing OpenShift environment, the following sequence must occure
* APB must be `built` into a docker image
* APB must be `pushed` into the docker registry that the broker is configured with
* The broker must `recognize` the new APB
* The service catalog must be `refreshed` to list the new APB for the user to consume

#### APB Build and Push to Registry
1. Download and install [APB Tool](https://github.com/ansibleplaybookbundle/ansible-playbook-bundle#installing-the-apb-tool)
1. Issue the `apb build` command in the APB's base folder (containing `apb.yml`)
1. Issue the `docker images` command to view the APB images
1. Re-tag the image if necessary to reflect the registry that the broker is configured with
    ```bash
    e.g.
    docker tag <image-ID> docker.io/<my-org>/<my-apb-name>
    ```
1. `docker push` push the image to your org/registry

#### Re-Deploy Broker 
1. Verify that the `refresh_interval` is something large (e.g. `24h`) by editing the broker's config map via WebUI or by running the `oc edit configmap -n ansible-service-broker` command.
    
    Search for the following section of the configmap and ...
    ```bash
    broker:
        dev_broker: True
        bootstrap_on_startup: true
        refresh_interval: "600s"
    ```
    Edit, so that the `refresh_interval` looks like
    ```
        broker:
        dev_broker: True
        bootstrap_on_startup: true
        refresh_interval: "24h"
    ```
    Then redeploy the broker pod in WebUI or by running the `oc rollout latest asb -n ansible-service-broker` command to pick up the configmap changes.

    The above changes ensure that for the 24 hours after you run APB push, testing the APB you've pushed should continue to work without having to continually re-push every

#### Refresh Service Catalog
To force the refresh of the APB's in the Service Catalog, you can relaunch the `controller-manager` pod of the service catalog, so that you won't have to wait the default time to refresh.  After the `controller-manager` pod has relaunched, refresh the WebUI.

### APB Deployment
Once the APB is presented in the WebUI you can deploy the APB by doing the follow steps:
1. Create a new project for the APB
1. Select one of the plans if multiple plans are provided
1. Enter all of the required parameter values
1. Select 'do not bind at this time' option, and deploy
1. Review the logs of the APB and verify that the ansible playbook finsihed without any failures. 

### AWS APB List
For each APB, do the following
  * What is it?
  * Pre-reqs?
  * Parameters - Explained
  * Bind Credentials returned
  * How to use
  * Example
    * Provisioning (include screen shots)
      * Bind
      * Test App (if any)

#### SNS - Simple Notification Service

#### SQS - Simple Queue Service 

#### Route 53

#### RDS 

