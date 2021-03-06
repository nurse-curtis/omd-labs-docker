# omd-labs-docker

OMD Labs Nightly (https://labs.consol.de/de/omd/index.html) on Docker with Ansible support.

Author: Simon Meggle, *simon.meggle at consol.de*

## Automated builds

Each image build gets triggered by the OMD Labs build system as soon as there are new packages of OMD available:

* https://hub.docker.com/r/consol/omd-labs-centos/
* https://hub.docker.com/r/consol/omd-labs-debian/
* https://hub.docker.com/r/consol/omd-labs-ubuntu/

The image already contains a "demo" site.

## Usage

### run the "demo" site

Run the "demo" site in OMD Labs Edition:

    # Centos 7
    docker run -p 8443:443 consol/omd-labs-centos
    # Ubuntu 16.04
    docker run -p 8443:443 consol/omd-labs-ubuntu
    # Debian 8
    docker run -p 8443:443 consol/omd-labs-debian

Use the Makefile to work with *locally built* images:

    # run a local image
    make -f Makefile.omd-labs-centos start
    # build a "local/" image without overwriting the consol/ image
    make -f Makefile.omd-labs-centos build
    # start just the bash
    make -f Makefile.omd-labs-centos bash

The container will log its startup process:

```
Config and start OMD site: demo
--------------------------------------
Data volume check...
--------------------------------------
 * [LOCAL]    /opt/omd/sites/demo/local
 * [LOCAL]    /opt/omd/sites/demo/etc
 * [LOCAL]    /opt/omd/sites/demo/var

Checking for Ansible drop-in...
--------------------------------------
Nothing to do (/root/ansible_dropin/playbook.yml not found).

omd-labs: Starting site demo...
--------------------------------------
Preparing tmp directory /omd/sites/demo/tmp...Starting gearmand...OK
Starting rrdcached...OK
Starting npcd...OK
Starting nagios...OK
Starting dedicated Apache for site demo...OK
Initializing Crontab...OK
OK
```

Notice the section "Data volume check". In this case there were no host mounted data volumes used. The "start.sh" script has renamed all `.ORIG` folders to the original name in case there are no mounted volumes.

### run a custom site

If you want to create a custom site, you have to build an own image:

* clone this repository, `cd` into the folder containg the Dockerfile, e.g. `omd-labs-centos`
* build a local image:
      export NEW_SITENAME=mynewsite; make -f Makefile.omd-labs-centos build    
* run the image:
      docker run -p 8443:443 local/omd-labs-centos

### Use data containers

#### Host mounted data folders

As soon as the container dies, all monitoring data (configuration files, RRD data, InfluxDB, log files etc.) are lost, too. To keep the data persistent, use host mounted volumes.

This command

      make -f Makefile.omd-labs-centos startvol

starts the container with three volume mounts:

* `./site/etc` => `$OMD_ROOT/etc`
* `./site/local` => `$OMD_ROOT/local`
* `./site/var` => `$OMD_ROOT/var`

On the very first start, this folders will be created on the host file system.
In that case, the `start.sh` populates them with the content of the original folders (`etc.ORIG, local.ORIG, var.ORIG`) within the container:

```
Config and start OMD site: demo
--------------------------------------
Data volume check...
--------------------------------------
 * [EXTERNAL] /opt/omd/sites/demo/local
 * [EXTERNAL] /opt/omd/sites/demo/etc
 * [EXTERNAL] /opt/omd/sites/demo/var

Checking for Ansible drop-in...
--------------------------------------
Nothing to do (/root/ansible_dropin/playbook.yml not found).

omd-labs: Starting site demo...
--------------------------------------
Preparing tmp directory /omd/sites/demo/tmp...Starting gearmand...OK
Starting rrdcached...OK
Starting npcd...OK
Starting nagios...OK
Starting dedicated Apache for site demo...OK
Initializing Crontab...OK
OK
```

On the next start the folders are *not* empty anymore and used as usual.




#### Start OMD-Labs with data volumes

To test if everything worked, simply start the container with

      make startvol

This starts the container with the three data volumes. Everything the container writes into one of those three folder, it will write it into the persistent file system.   

(`make startvol` is just a handy shortcut to bring up the container. In Kubernetes/OpenShift you won't need this.)  

## Ansible drop-ins

For some time OMD-Labs comes with **full Ansible support**, which we can use to modify the container instance *on startup*. **How does this work?**

### start sequence
By default, the OMD-labs containers start with the CMD `/root/start.sh`. This script

* checks if there is a `playbook.yml` in `$ANSIBLE_DROPIN` (default: `/root/ansible_dropin`, changeable by environemt). If found, the playbook is executed. It is completely up to you if you only place one single task in `playbook.yml`, or if you also include Ansible roles. (with a certain point of complexity, you should think about a separate image, though...)
* starts the OMD site "demo" & Apache as a foreground process

### Include Ansible drop-ins

Just a folder containing a valid playbook into the container:

    docker run -it -p 8443:443 -v $(pwd)/my_ansible_dropin:/root/ansible_drop consol/omd-labs-debian

### Debugging

If you want to see more verbose output from Ansible to debug your role, add the environment variable `ANSIBLE_VERBOSITY`:

    docker run -it -p 8443:443 -e ANSIBLE_VERBOSITY="-vv" -v $(pwd)/my_ansible_dropin:/root/ansible_drop consol/omd-labs-debian
