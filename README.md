# TFM_Openstack_Tacker_OpenDayLight
Integration of Openstack and OpenDayLight together with the Tacker component to deploy a cloud computing platform as an IaaS solution

![Integración Openstack - Opendaylight](https://raw.githubusercontent.com/vquilon/TFM_Openstack_Tacker_OpenDayLight/master/img/TFM.png)

# SetUp Environment
## Requirements
Need at least a single node with Linux OS, like an Ubuntu 18.04 Bionic.
* RAM: at least 16GB.
* Storage: 50GB.
* CPU: at least 4 Cores.
## Increase RAM memory with SWAP file
You can increment the RAM with 8GB of swal file. execute the next commands to create swal file. (The example shows how to get 16GB with a single node of 8GB of RAM):

```
sudo swapoff /swapfile
sudo fallocate -l 8G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
sudo swapon /swapfile
```

## Updates and installations
Next we need,
* git command to download all Openstack components repositories.
* Virtualenv, the environment where all python components will run.
* net-tools, basic CLI tools for networking interaction (ex: `ifconfig`).

```
sudo apt install git
sudo apt install virtualenv
sudo apt install net-tools
```

## OVS Installation (openvswitch)
```
sudo apt-get install -y autoconf libtool git dh-autoreconf dh-systemd software-properties-common 
sudo apt-get install -y linux-image-generic libssl-dev openssl build-essential fakeroot graphviz python-all python-qt4
sudo apt-get install -y python-twisted-conch dkms
```
Clone the repository of openvswitch [Github](https://github.com/openvswitch/ovs.git)
```
git clone https://github.com/openvswitch/ovs.git
cd ovs
git checkout -b v2.9.2
```
> Before build the packages .deb we have to fix a bug known inside the make file. Replace the file for the following corrected content.

**lib/automake.mk**
```
 		openssl dhparam -C -in $(srcdir)/lib/dh2048.pem -noout &&	\
 		openssl dhparam -C -in $(srcdir)/lib/dh4096.pem -noout)	\
 		| sed 's/\(get_dh[0-9]*\)()/\1(void)/' > lib/dhparams.c.tmp &&  \
+++ 	sed -i '/\(get_dh[0-9]*\)(void)/s/^static//' lib/dhparams.c.tmp && \
		mv lib/dhparams.c.tmp lib/dhparams.c
else
lib_libopenvswitch_la_SOURCES += lib/stream-nossl.c
```
Create the installation packages
```
sudo DEB_BUILD_OPTIONS='parallel=8 nocheck' fakeroot debian/rules binary
cd ..
sudo mkdir -p /vagrant/ovs_debs
sudo cp ./libopenvswitch_*.deb ./openvswitch-common*.deb ./openvswitch-switch*.deb /vagrant/ovs_debs/
sudo dpkg -i ./libopenvswitch_*.deb ./openvswitch-datapath-dkms* ./openvswitch-common* ./openvswitch-switch* ./python-openvswitch*
```
Now restart the service
```
service openvswitch-switch restart
```

## Increment watchers number (Error related with the behaviour in linux services)
We must be increased so that Openstack logs do not reach the limit and **devstack** cannot be displayed.
```
echo fs.inotify.max_user_watches=524288 | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```
# Devstack installation
We are going to proceed with the installation of Openstack Rocky and OpenDaylight Oxygen

## `stack` user creation
```
sudo useradd -s /bin/bash -d /opt/stack -m stack
echo "stack ALL=(ALL) NOPASSWD: ALL" | sudo tee /etc/sudoers.d/stack
```
Use the `stack` user
```
sudo su - stack
```
Clone the **devstack** project
```
git clone https://git.openstack.org/openstack-dev/devstack -b stable/rocky
```
## Increment the timeout service
It is possible to increase the timeout associated with the services when they are started, it is advisable to carry out this configuration since if the linux system is being virtualized, it may cost the CPU to process the deployment of devstack and need more time so that it does not abort the execution devstack automatic.
**/opt/stack/devstack/stackrc**
```
# Service startup timeout
SERVICE_TIMEOUT=${SERVICE_TIMEOUT:-300}
```
## Tacker Rocky branch BUG found
There is a bug in a configuration file in the Rocky version of the Tacker component. To solve it go to the file **/opt/stack/tacker/etc/config-generator.conf** and modify the following line.
> Before you have to clone the Tacker repository, [https://git.openstack.org/openstack/tacker](https://git.openstack.org/openstack/tacker)
```
git clone https://git.openstack.org/openstack/tacker -b stable/rocky
```
**/opt/stack/tacker/etc/config-generator.conf**
```
-namespace = tacker.vnfm.infra_drivers.kubernetes.kubernetes
+namespace = tacker.vnfm.infra_drivers.kubernetes.kubernetes_driver
```
## Virtualenv initialization often BUG
To fix it clone the repository before running **stack.sh** and launching virtualenv
```
git clone https://git.openstack.org/openstack/requirements.git -b stable/rocky
```
```
virtualenv /opt/stack/requirements/.venv/
```

# Devstack execution
> Before running devstack, you have to modify the **local.conf** file to the needs of the infrastructure you want to deploy.
```
git clone https://git.openstack.org/openstack/devstack -b stable/rocky
```
Copy the local.conf file located in the devstack folder of this repository and place it in **/opt/stack/devstack/local.conf**
```
cp local.conf /opt/stack/devstack/local.conf
```
Execute **DEVSTACK**.
```
cd devstack
./stack.sh
```
# Deployment check
As an example of a scenario to see that the unfolding works correctly, it can be verified with the following steps,
* Access Horizon and DLUX
	* Horizon: [http://HOST_IP/dashboard](http://HOST_IP/dashboard)
	* DLUX: [http://HOST_IP:8181/index.html](http://HOST_IP:8181/index.html)
	> The IP is configured in **local.conf** in the ``HOST_IP`` and ``SERVICE_HOST`` parameters
* In Networks, check if the networks created by default are present
	* net0
	* net1
	* net_mgmt
## Building a demo stage
Infrastructure deployment can be verified using a scenario provided by the Tacker component itself. Documentation is available at [Despliegue de un escenario de VNFs con Tacker](https://docs.openstack.org/tacker/latest/user/nsd_usage_guide.html).

## Network topology of the deployed scenario
![Escenario demo del componente Tacker](https://raw.githubusercontent.com/vquilon/TFM_Openstack_Tacker_OpenDayLight/master/img/topologia_red_escenario_demo.png)
