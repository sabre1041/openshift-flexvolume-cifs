# openshift-flexvolume-cifs

FlexVolume driver for access CIFS based shares

## Overview

[FlexVolume's](https://docs.openshift.com/container-platform/3.11/install_config/persistent_storage/persistent_storage_flex_volume.html) provide a method for users to create their own drivers for adding options for utilizng persistent storage in an OpenShift and Kubernetes environment. This driver allows for the creation of a [PersistentVolume](https://docs.openshift.com/container-platform/3.11/architecture/additional_concepts/storage.html) or direct mount to access storage that is exposed by a CIFS share.

## Setup

Utilize either one of the following methods to configure FlexVolume support in your environment

### Manual setup

Installation and configuration of the CIFS FlexVolume driver can be completed manually or in an automated fashion using [Ansible](https://www.ansible.com/). 

1. Install required packages

```
yum install cifs-utils
```

2. Install the driver

The [cifs](flexvolume-driver/cifs) driver needs to be placed on all Master instances as well as any node that applications leveraging the FlexVolume driver will be mounted. The drivers that are used to support FlexVolume's are placed within the `/usr/libexec/kubernetes/kubelet-plugins/volume/exec/` directory. Within this directory, drivers are stored by vendor name and driver name in the format `<vendor>~<driver>`. Tehe vendor for this driver is `openshift.io` and the driver name is called `cifs`. Create a directory called `/usr/libexec/kubernetes/kubelet-plugins/volume/exec/openshift.io~cifs` and place the cifs driver executable within it

3. Restart the OpenShift Services

On the OpenShift Master's, restart the controllers

```
master-restart controllers
```

On all nodes the driver was configured on, restart the Atomic Node service

```
systemctl restart atomic-openshift-node
```

### Ansible based environment setup

The installation process for the FlexVolume driver can be automated through the use of an Ansible playbook located in the [ansible](ansible) directory.

### Configure the inventory

An example inventory file called [inventory](ansible/inventory) is available in the [ansible](ansible) directory. The inventory groups align with the group names utilized in the OpenShift installation meaning that inventories created for existing environments can be utilized. Otherwise, fill out the _masters_ and _nodes_ with the instances that are desired to be configured with the CIFS FlexVolume driver.

## Implementation

The following steps will not only walk through some of the available options provided by the CIFS driver, but showcase how one can utilize it in your environment.

### Secrets

In many cases, access to CIFS shares will be protected and require authentication. Credentials to access CIFS volumes can be placed in a secret and then referenced in the FlexVolume definition. 

A secret can either be manually created or an OpenShift template can be processed to create a secret. 

To manually create the required secret, create a file called `flexvolume-cifs-secret.yml` with the following contents

```
apiVersion: v1
kind: Secret
metadata:
  name: cifs-credentials
type: openshift.io/cifs
stringData:
  password: <username>
  username: <password>
```

Replace `<username>` and `<password>` with the actual values

Create or utilize an existing namespace and apply the previously created secret file

```
oc apply -f secret.yml
```

Instead of manually creating a secret file and then applying it to the cluster, an existing template file called [credentials-secret-template.yml](examples/cifs-secret-template.yml) can be utilized from the examples folder.

The template requires that a username and password be provided as parameters. 

Execute the following command to instantiate the template

```
oc process -p USERNAME=<username> -p PASSWORD=<password> -f examples/cifs-secret-template.yml | oc apply -f-
```

A new secret called _cifs-credentials_ will be created

_NOTE_: In addition to username and password, the domain associated with the CIFS share can also be specified. If desired, add the `domain` key to the secret.

A _credentials_ file will be located one directory above the location where the volume is mounted in the format `.<MOUNT_NAME>.credentials

### Security Context Contraints

By default, all pods created by typical end users make use of the _restricted_ [Security Context Constraint](https://docs.openshift.com/container-platform/3.11/admin_guide/manage_scc.html). FlexVolumes are not one of the allowed volume types. Instead of modifying the existing _restricted_ SCC, it is recommended that a custom SCC be created instead. An example of an SCC based upon the restricted SCC with support for FlexVolume's is provided in the examples folder called [restricted-flexvolume-scc.yml](examples/restricted-flexvolume-scc.yml).

As a cluster administrator, apply the SCC to the cluster.

```
oc apply -f examples/restricted-flexvolume-scc.yml
```

With the SCC in place, grant the service account that will be used by applications access to the newly created SCC. The following example describes how to grant the _default_ service account to the custom SCC.

```
oc adm policy add-scc-to-user restricted-flexvolume -z default
```

### FlexVolume Definition

As previously described, FlexVolume definitions can be placed either on a PersistentVolume or directly on the object. An example is defined below:

```
flexVolume:
  driver: openshift.io/cifs
  fsType: cifs
  options:
    mountOptions: 'file_mode=0770,dir_mode=0770'
    networkPath: //cifsserver/cifsShare
  secretRef:
    name: cifs-credentials
  name: cifs
```

Be sure to configure the `networkPath` option with the location of the CIFS sahre. In addition, it is recommended (almost required) that the above `mountOptions` be specified in order to comply with OpenShift's security model.

_Note_: The security context mount option `context=system_u:object_r:svirt_sandbox_file_t:s0` is used by default in order to grant containers access to the file system the share is mounted on where SELinux is enabled. You may override this with a different context, however it is no recommended.

## Existing Implementations

The contents of this repository are not the first attempt at a CIFS FlexVolume driver. However, due to the combination of OpenShift's security requirements, as well as the architecture of the _controller-manager_, it became necessary for another implementation to be created. Many thanks are due to the following implementations:

* [fstab/cifs](https://github.com/fstab/cifs)
* [Azure CIFS/SMB FlexDriver for Kubernetes](https://github.com/Azure/kubernetes-volume-drivers)

