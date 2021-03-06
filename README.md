## About

This project is largely based on of a great repository made by [Roland Huß](https://github.com/rhuss):
[Project31/ansible-kubernetes-openshift-pi3](https://github.com/Project31/ansible-kubernetes-openshift-pi3).

Almost everything here (included some parts of this README) is copied from that repository, I changed just a couple of things in order to better handle my custom configurations.

### Features

* From zero to a running cluster with a single Ansible playbook
* Kubernetes 1.8 with Docker 17
* WiFi networking and Flannel as CNI
* Cluster aware storage available out of the box using GlusterFS (replicated on every node)
* Dashboard and Heapster deployed by default along with Traefik (a reverse proxy that exposes your containers to the outside world)
* Manual deployments for the Docker registry, MySQL, whoami and Lavagna.

## Setup the hardware

Most of the installation is automated by using [Ansible](https://www.ansible.com/).
Thanks to [Hypriot](https://github.com/hypriot/image-builder-rpi/releases/latest) images a complete headless setup is possible.

1. Download the latest Hypriots image and store it as `hypriot.zip`:
```
curl -L https://github.com/hypriot/image-builder-rpi/releases/download/v1.5.0/hypriotos-rpi-v1.5.0.img.zip
```
2. Install Hypriot's [flash](https://github.com/hypriot/flash) installer script. Follow the directions on the installation page.

3. Insert you Micro-SD card in your Desktop computer (via an adapter possibly) and run
```
flash --hostname n1 --ssid "mysid" --password "secret" hypriotos-rpi-v1.5.0.img.zip
```
   You will be asked to which device to write. Check this carefully, otherwise you could destroy your Desktop OS if selecting the the wrong device. Typically its something like `/dev/disk2` on OS X, but depends on the number of hard drives you have.
4. Repeat step 2. to 3. for each Micro SD card. Please adapt the hostname before each round to **n2**, **n3**, **n4**.

## Configure the network

It is now time to configure your WLAN router. This of course depends on which router you use. The following instructions are based on a [TP-Link TL-WR802N](http://www.tp-link.de/products/details/TL-WR802N.html) which is quite inexpensive but still absolutely ok for our purposes since it sits very close to the cluster and my notebook anyway.

First of all you need to setup the SSID and password. Use the same credentials with which you have configured your images.

The addresses I have chosen are:

| IP                                    | Device          |
| ------------------------------------- | --------------- |
| `192.168.23.1`                        | WLAN Router     |
| `192.168.23.181` ... `192.168.23.184` | Raspberry Pis   |


You should be able to SSH into every Pi with user *pirate* and password *hypriot*. Internet access from the nodes is mandatory for setting up the nodes with Ansible.

After that you checked every Raspberry you can configure the router using these steps detailed by [Sergio Sisternes](https://twitter.com/sesispla):

* Go to *DHCP > Client list* and take note of the MAC addresses
* Go to *DHCP > Address reservation*, and set an static (Reserved) IP address to each Raspberry
* Go to *Quick Setup* and configure the router in the *hotspot mode* so it can connect itself to another router and share Internet with the Raspberries

## Ansible playbooks

After this initial setup is done, the next step is to initialize the base system with Ansible. You will need Ansible 2 installed on your desktop (e.g. `brew install ansible` when running on OS X)

### Ansible configuration

1. Checkout the Ansible playbooks:

        git clone https://github.com/SitoCH/rpi-kubernetes-cluster.git rpi-kubernetes-cluster
        cd rpi-kubernetes-cluster

2. Copy over `hosts.example` and adapt it to your needs

        cp hosts.example hosts
        vi hosts

   There are three groups:

   * **pis** contains all members of your cluster where one is marked as "master" in the field `host_extra`. This group will be added to every node in its `/etc/hosts`. **It is important that one host is marked as "master", since the playbooks rely on this host alias for accessing the API server**.
   * **master** IP address of the Master
   * **nodes** All nodes which are not Master
   * **volumes** Volumes to create on GlusterFS

### Configure the cluster

Run the deployment with the command:

    ansible-playbook -k -i hosts setup.yml

### Update the nodes

After some time this playbook may be useful to update the nodes:

    ansible-playbook -k -i hosts apt-upgrade-cluster.yml

### Deploy default applications

Once that the cluster is up and running this playbook will deploy GlusterFS endpoints, Heapster, Dashboard and Traefik:

    ansible-playbook -k -i hosts deployments.yml

#### Traefik

Traefik is a reverse proxy that exposes your containers to the outside world. The management dashboard is available at `http://master-node:8080/`

## Manual deployments

For convenience useful deployments can be found in the directory *manual-deployments*, please note that you must install the Registry in order to use other manual deployments because the images will be centrally cached.

### Registry

This container installs a Docker registry that acts as a pass-trough cache, this way only the first node will download an image from Internet and all the other requests will be served from "inside" the cluster.

    kubectl --kubeconfig run/admin.conf create -f manual-deployments/registry/registry.yml

### Who am I

Simple container that deploys on 3 nodes a website that prints it's container ID.

Thanks to Traefik you can access it on `http://master-node/whoami/`

    kubectl --kubeconfig run/admin.conf create -f manual-deployments/whoami/whoami.yml

### MySQL

MySQL database server with persistent storage.

    kubectl --kubeconfig run/admin.conf create -f manual-deployments/mysql/mysql.yml

### Lavagna

Lavagna is an open-source issue/project management tool designed for small teams (another open source project of mine made with 2 friends).

    kubectl --kubeconfig run/admin.conf create -f manual-deployments/lavagna/lavagna.yml



## Acknowledgements

[Roland Huß](https://github.com/rhuss) for doing the heavy lifting and creating a project ready to use and easy to deploy.

[Lucas Käldström](https://github.com/luxas) for porting Kubernetes to the Raspberry Pi / ARM.

[Sergio Sisternes](https://twitter.com/sesispla) for providing a great guide on how to setup the WiFi router.