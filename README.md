# TFM_Openstack_Tacker_OpenDayLight
Integration of Openstack and OpenDayLight together with the Tacker component to deploy a cloud computing platform as an IaaS solution
![Integración Openstack - Opendaylight](https://raw.githubusercontent.com/vquilon/TFM_Openstack_Tacker_OpenDayLight/master/img/TFM.png)
# Preparación del entorno
## Requisitos
Se necesita al menos un ordenador con sistema Linux, como por ejemplo Ubuntu 18.04 Bionic.
* RAM: Mínimo 16 GB de RAM.
* Almacenamiento: 50GB.
* CPU: Mínimo 4 Cores.
## Añadir RAM con swapfile
Es posible añdir 8GB de RAM a la configuración de tu sistema si solo se dispone de otros 8GB.
```
sudo swapoff /swapfile
sudo fallocate -l 8G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
sudo swapon /swapfile
```
## Actualización e instalaciones básicas
A continuación se necesitarán,
* GIT para la descarga de los diferentes componentes de Openstack
* Virtualenv, entorno donde se instalan los paquetes
* net-tools, Paquete interfaz CLI para la ejecución por ejemplo de `ifconfig`

```
sudo apt install git
sudo apt install virtualenv
sudo apt install net-tools
```

## Instalación de OVS
```
sudo apt-get install -y autoconf libtool git dh-autoreconf dh-systemd software-properties-common 
sudo apt-get install -y linux-image-generic libssl-dev openssl build-essential fakeroot graphviz python-all python-qt4
sudo apt-get install -y python-twisted-conch dkms
```
Después se clona el proyecto de [Github](https://github.com/openvswitch/ovs.git)
```
git clone https://github.com/openvswitch/ovs.git
cd ovs
git checkout -b v2.9.2
```
> Antes de construir los paquets .deb hay que arreglar un bug que hay en el fichero Make.

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
Creamos los paquetes de instalación
```
sudo DEB_BUILD_OPTIONS='parallel=8 nocheck' fakeroot debian/rules binary
cd ..
sudo mkdir -p /vagrant/ovs_debs
sudo cp ./libopenvswitch_*.deb ./openvswitch-common*.deb ./openvswitch-switch*.deb /vagrant/ovs_debs/
sudo dpkg -i ./libopenvswitch_*.deb ./openvswitch-datapath-dkms* ./openvswitch-common* ./openvswitch-switch* ./python-openvswitch*
```

```
service openvswitch-switch restart
```
## Aumentar el número de Watchers
Se deben aumentar para que los logs de Openstack no llegen al limite y no pueda desplegarse **devstack**.
```
echo fs.inotify.max_user_watches=524288 | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```
# Instalación de Devstack
Se va a proceder con la instalación de Openstack Rocky y OpenDaylight Oxygen

## Creación de usuario stack
```
sudo useradd -s /bin/bash -d /opt/stack -m stack
echo "stack ALL=(ALL) NOPASSWD: ALL" | sudo tee /etc/sudoers.d/stack
```
```
sudo su - stack
```

Clonamos el proyecto **devstack**
```
git clone https://git.openstack.org/openstack-dev/devstack -b stable/rocky
```
## Aumentar el timeout de servicios
Es posible aumentar el timeout asociado a los servicios cuando se inician, es recomendable realizar esta configuración ya que si se esta virtualizando el sistema linux, puede que le cueste a la CPU procesar el despligue de devstack y necesite más tiempo para que no aborte la ejecución automática de devstack.
**/opt/stack/devstack/stackrc**
```
# Service startup timeout
SERVICE_TIMEOUT=${SERVICE_TIMEOUT:-300}
```
## Bug en la rama de Tacker Rocky
Existe un bug en un fichero de configuración en la versión Rocky del componente de Tacker. Para solucionarlo ir al fichero **/opt/stack/tacker/etc/config-generator.conf** y modificar la siguiente linea.
> Antes habrá que clonar el repo de Tacker, [https://git.openstack.org/openstack/tacker](https://git.openstack.org/openstack/tacker)

```
git clone https://git.openstack.org/openstack/tacker -b stable/rocky
```
**/opt/stack/tacker/etc/config-generator.conf**
```
-namespace = tacker.vnfm.infra_drivers.kubernetes.kubernetes
+namespace = tacker.vnfm.infra_drivers.kubernetes.kubernetes_driver
```
## Fallo a menudo en la inialización del virtualenv
Para solucionarlo clonar el repositorio antes de ejecutar **stack.sh** y lanzar virtualenv
```
git clone https://git.openstack.org/openstack/requirements.git -b stable/rocky
```
```
virtualenv /opt/stack/requirements/.venv/
```

# Ejecutar Devstack
> Antes de ejecutar devstack hay que modificar el fichero **local.conf** a las necesidades de la infraestructura que se quiere montar.
```
git clone https://git.openstack.org/openstack/devstack -b stable/rocky
```
Copiar el fichero local.conf ubicado en la carpeta devstack de este repositorio y ubicarlo en **/opt/stack/devstack/local.conf**
```
cp local.conf /opt/stack/devstack/local.conf
```
Por ultimo ejecutar **DEVSTACK**.
```
cd devstack
./stack.sh
```
# Comprobación del despliegue
Como ejemplo de escenario para ver que funciona correctamente el deslpliegue se puede comprbar con los siguientes pasos,
* Acceder a Horizon y DLUX
	* Horizon: [http://HOST_IP/dashboard](http://HOST_IP/dashboard)
	* DLUX: [http://HOST_IP:8181/index.html](http://HOST_IP:8181/index.html)
	> La IP es la configurada en **local.conf** en los parametros ``HOST_IP`` y ``SERVICE_HOST``
* En Redes, comprobar si están las redes creadas por defecto
	* net0
	* net1
	* net_mgmt
## Construccion de un escenario demo
Se puede comprobar el despliegue de la infraestructura mediante un escenario que proporciona como ejemplo el propio componente de Tacker. La documentación esta disponible en [Despliegue de un escenario de VNFs con Tacker](https://docs.openstack.org/tacker/latest/user/nsd_usage_guide.html).

## Topología de la red del escenario desplegado
![Escenario demo del componente Tacker](https://raw.githubusercontent.com/vquilon/TFM_Openstack_Tacker_OpenDayLight/master/img/topologia_red_escenario_demo.png)
