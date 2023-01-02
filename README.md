# graylog
How to get Graylog running in your home using Docker


I have spend several nights in trying to get Graylog running at home. There are a lot of manuals out there, some very basic, some slightly outdated. I got a lot of help from the Graylog-Community-Forum and the Docker-Community-Forum. Most of the following inputs are based on their knowledge and advise.
So here is the summary on how I did it.


## Background
* Server: I got my hands on an old _HP EliteDesk 800 G2_ with 8GB RAM and 128GB disk space. This will be my server, running _Ubuntu Server 22.04.1 LTS_.
* Network: I have a Unifi environment at home. The network is segmented into different VLANs. The Ubuntu and Graylog should be in _VLAN70_. Other containers will be in other VLANS. (e.g. HomeAssisnant, AdGuard, OpenVPN, ...)

## Preparation
### promiscuous mode
As I am working with different VLANs, the network card of my server has to be in _promiscuous mode_. [Docker: Use macvlan networks](https://docs.docker.com/network/macvlan/)
Put this code in your /etc/rc.local. [askubuntu](https://askubuntu.com/questions/430355/configure-a-network-interface-into-promiscuous-mode)
```
ifconfig eth1 up
ifconfig eth1 promisc
```
Make sure to set the right network name. (My network is called _eno1_.)

### prepare VLANs in docker
Later on, I will reference the VLAN in the compose.yaml. This will tell the container to use that network. The following command basically creates a docker-specific DHCP for that VLAN. Make sure that you do not get dublicate IPs in your network!

```
# Server-Netz (VLAN 70)
docker network create -d macvlan \
    --subnet=192.168.70.0/24 \
	--ip-range=192.168.70.127/26 \
    --gateway=192.168.70.1 \
	--aux-address="Ubuntu-Docker-Server=192.168.70.2" \
    -o parent=eno1.70 macvlan70
```
Docker manual on Macvlan networks [802.1q trunk bridge mode](https://docs.docker.com/network/macvlan/)
:warning: A macvlan will make your container puplic to your network. You will see the container in your router, you will have to ristrict access by using your routers/firewalls means.
:warning: there is the option to use ipvlans as well. The image within the documentation looked like what I wanted, but I never got it running properly.
![IPvlan 802.1q trunk L2 mode example usage](https://docs.docker.com/network/images/vlans-deeper-look.png)


### 802.1q trunk switch port
My server is wired to a switch. Because I am using different VLANs, the switch port has to be configured, so that the respective traffic can be received on that switch port. In Unifi, this is done like this: [A non-expert Guide to VLAN and Trunks in Unifi Switches](https://community.ui.com/questions/A-non-expert-Guide-to-VLAN-and-Trunks-in-Unifi-Switches/7462245c-95a7-455e-a711-209f44e194cb)
* create a _switch port profile_
![Unifi_switchportprofile](/images/Unifi_switchportprofile.png)
* apply profile to switch port
![Unifi_switchport](/images/Unifi_switchport.png)

:warning: Please note: I want to have my server and some of the Docker containers in the same VLAN. Unfortunately, I was not able to get this working. (Still open discussion in Docker forum: [How to set host and containes in same vlan](https://forums.docker.com/t/how-to-set-host-and-containes-in-same-vlan/133416))

### Folders for persisting data
To be able to persist data, you need to create some folders and set the needed permissions.

#### create user _graylog_ with id _1100_
When you start your graylog container in docker, it will be using a user _graylog_ with id _1100_. You have to create this user in your ubuntu.

```
sudo useradd -u 1100 graylog
```

#### create _graylog_data_
Create a folder with the name _graylog\_data_. Do this in a path with easy access. I have chosen this one:
```
mkdir /home/uadmin/Docker/Graylog/graylog_data
```
As my server is running in my own network, the server itself has very limited access to the internet, I gave access to all users:
```
sudo chmod -R a+rwx /home/uadmin/Docker/Graylog/graylog_data
```

#### create _es\_data_
Elasticsearch needs a folder as well to persist data.
```
mkdir /home/uadmin/Docker/Graylog/es_data
```
Change access rights:
```
sudo chmod -R a+rwx /home/uadmin/Docker/Graylog/es_data
```

#### create _mongo\_data_
```
mkdir /home/uadmin/Docker/Graylog/mongo_data
```
```
sudo chmod -R a+rwx /home/uadmin/Docker/Graylog/mongo_data
```


## Docker compose file
I found many different versions of the docker compose file for graylog. Mine is based on the official documentation: [Persisting Data](https://go2docs.graylog.org/5-0/downloading_and_installing_graylog/docker_installation.htm#PersistingData). _Version 2_ is rather outdated, so I went over the official docker [_Compose specification_](https://docs.docker.com/compose/compose-file/) and adopted it to the newest version (as of december 2022).
In the following, I will highlight one or two points. To fully understand the file, I recommend to have a closer look at the specifications of Docker and Graylog.

And here it is, my [**compose.yaml**](compose.yaml).

### restart
There is an option, where you can define, in what cases docker should performe a restart of a container.
```
    ...
    restart: unless-stopped
    ...
```
Docker compose specification on [restart](https://docs.docker.com/compose/compose-file/#restart)

### volumes
There is a big difference between _volumes_ and _binds_ - to be honest, I did not fully understand. _binds_ are folders sitting somewhere on your host, that can be mapped into your docker container. Volumes are sitting at /var/lib/docker/volumes/ on your host and can only be accessed with sudo-rights. Volumes can be managed via docker (e.g. delete all un-used volumes), where as binds can be put into a path of your liking. What is better? Well, I don't no. If you need to frequently access those files to do some configuration, use a bind like this:

```
    volumes:
      - /home/uadmin/Docker/Graylog/mongo_data:/data/db
      
      
volumes:
  mongo_data:
    driver: local
  es_data:
    driver: local
  graylog_data:
    driver: local
    
```
Docker compose specification on [volumes](https://docs.docker.com/compose/compose-file/#volumes)
Docker compose specification on [Volumes top-level element](https://docs.docker.com/compose/compose-file/#volumes-top-level-element)


### networks
I defined two networks. One is the 
```
  graylog:
    ...
    networks:
      macvlan70:
        ipv4_address: 192.168.70.3
      graylog_backend:
        ipv4_address: 10.10.10.2
    ...
```




