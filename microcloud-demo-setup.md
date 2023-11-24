# Microcloud setup

## Requirements
  - Minimum Three nodes I used two xilinx boards kv260 and one mediatek board they were all running UC22 image.
  - Need two separate network interfaces on each board . These interfaces need not be physical they can be virtual interfaces.
  - One interface will act as management interface. This is how nodes communicate at the beginning to form the cluser.
  - Second inetrface will be used for microcloud's internal traffic and it is required that  this interface of every node (that want to
    be part of cluster)  to be connected to gateway. There should not be dhcp server in this network. This is called as **uplink network**.
    and it is L2 network per se.
  - Each node can have a disk for local storage. ::zfs .
  - Each node need to have a disk if you want to have distributed storage. ::ceph .
  - Please ensure that all your nodes will have distinct hostnames .  

## Cleaning Up previous microcloud setup related snaps
**Note-:**
This is also the  most common thing reuqired currently to handle any kind of error
during microcloud setup.

```
snap remove --purge lxd 
snap remove --purge microovn 
snap remove --purge microceph 
snap remove --purge microcloud

```

## Bring the network interfaces Up for the distrib netwroking to work 

These commands should be adapted as per network interface names

```
ip link set up eth0
ip link set up enx68e43b306586 
ip link set up enxc84d44201124 

```


## These steps are just allternative to direct install
```
snap download lxd --revision=25755
snap download microcloud --channel=latest/edge
snap download microovn 
snap download microceph

snap ack lxd_25755.assert && snap ack microovn_247.assert  && snap ack microceph_554.assert && snap ack microcloud_542.assert 
snap install lxd_25755.snap microcloud_542.snap microovn_247.snap microceph_554.snap

```
## Install the snaps

```
snap install lxd
snap install microovn --channel=22.03/stable
snap install microceph --channel=quincy/stable
snap install microcloud --channel=latest/stable

```
## initialize microcloud cluster

```
microcloud init

```
- This may ask few questions
  - interface IP to search for nodes.
  - Selecting nodes based on search results.
  - Do you want to setup local storage?
  - Do you want to set up distrib storage?
  - Do you want to setup distrib networking-: Please ensure that all the interfaces connected to Uplink network should be up
  - Select CIDR gateway for uplink network -: This is the gateway IP address .
- Once qiestions has been answered it takes few seconds(few minutes) to form microcloud cluster.
- you can use commands like below to see the status
  ```
  microcloud cluster list
  lxc cluster list
  microceph cluster list
  miroovn cluster list
  ```

Once the cluster is ready you can install lxd containers or VMs


## Creating network forward for UPLINK network of your microcloud setup

Following are the steps

1. ``` lxc network edit UPLINK ```

config should look something like as shown below

```
config:
  ipv4.gateway: 192.168.3.254/24
  ipv4.ovn.ranges: 192.168.3.30-192.168.3.70
  ipv4.routes: 192.168.3.0/24
  volatile.last_state.created: "false"
description: ""
name: UPLINK
type: physical
used_by:
- /1.0/networks/default
managed: true
status: Created
locations:
- cobuntu
- mtkubuntu
```
2. ``` lxc network forward create default 192.168.3.31 ```
3. ``` lxc network forward edit default  192.168.3.31 ```

Please ensure that config will look something like as given below-:

```
description: My first forward
config:
  target_address: 10.80.130.2
ports: []
listen_address: 192.168.3.31
location: ""
```
4. stop and start the instance again.



## Running some workload on Microcloud

- This particular example demonstrate a workload where webservices for controlling matter home automation devices running under lxc containers.

- One web service for commissioning matter devices.

- Second web service for controlling matter devices.

- These web services are part of a snap and the code is [here](https://github.com/prash813/matter-mqtt-demo-service.git)

- These services are meant to communicate to matter [chip-tool](https://snapcraft.io/chip-tool) running on RPi board to manage smart home matter devices
- For the communication glue there is mqtt broker.
- mqtt broker runs inside the container on microcloud.
- The web services running in container publish their commands on the mqtt.
- RPI subscribes to these commands and then execute the chip tool for controlling matter devices.
- mqtt subscription part and executing chip-tool is part of a snap , code is [here](https://github.com/prash813/matter-mqtt-demo-client.git).







# SOME SETUP OF MATTER-HAUTO-WEB-UI DEMO 
This functionality is provided by [matter-hauto-demo server side snap](https://github.com/prash813/matter-mqtt-demo-service.git) 
1. Dependent on core22 snap.
2. Make connections. 
``` snap connect  matter-hauto-demo:network-control ```
3. Make sure that you copy following matter device config files to snap dirs. Please note that these are just sample json files for configs of the matter 
   devices which are part of demo.  
``` cp devicelist1.json /var/snap/matter-hauto-demo/current/mnt/devicelist.json ```
``` cp opmodes1.json /var/snap/matter-hauto-demo/current/mnt/opmodes.json ```
4 Make sure  to set forwardedip configuration of snap e.g.
 ```
 snap set matter-hauto-demo forwardedip=192.168.3.31
```
The ip used for the above variable is based on network forward created few steps before.

5. Install mqtt and core18 snaps. Make sure you edit the config of mqtt at location
```
/var/snap/mosquitto/common/mosquitto.conf

```
with Following config items


```
persistence false
allow_anonymous true
listener 1883 10.80.130.2

```
listener address is IP address that lxc container gets from lxd networking framework like ovn or ubuntu fan.

6. Run matter-hauto-demo.matterdemo (if this is not the service in a [matter-hauto-demo server side snap](https://github.com/prash813/matter-mqtt-demo-service.git)).


# SETUP ON MATTER HUB DEVICE (RPi board) 

## OTBR-snap setup
1. connect all the interfaces
2. set appropriate value for infra-iface
3. please execute following steps
```
	otbr-snap.ot-ctl dataset init new
	otbr-snap.ot-ctl dataset commit active
	otbr-snap.ot-ctl ifconfig up
	otbr-snap.ot-ctl thread start
	sleep 5
	otbr-snap.ot-ctl state
	otbr-snap.ot-ctl netdata show
	otbr-snap.ot-ctl ipaddr
	otbr-snap.ot-ctl dataset active -x


```
## setting matter mqtt client
This functionality is provided by [matter-hauto-demo client side snap](https://github.com/prash813/matter-mqtt-demo-client.git).

1. Since this hub is on uplink network of microcloud which does not have dhcp server, network config need to be set up manually.

```
ip address add 192.168.3.5/24 dev eth0
ip link set up eth0
ip route add  192.168.3.254 dev eth0
ip route add default via 192.168.3.254

```
2. install and set chip-tool properly.

3. copy devicelist.json and opmodes.json to ``` /var/snap/matter-hauto-demo/current/mnt/ ``` .
4. copy pemcerts to ``` /var/snap/matter-hauto-demo/common/pemcerts/ ``` .

5. Before executing mqtt matter client on RPi please ensure that following configs has been set.
```
Key           Value
debug         false
mqttbrokerip  192.168.3.31
nwiface       eth0
```
6. For nanoleaf bulbs or other thread devices PAA/API certificate assets need to be part of [matter-hauto-demo client side snap](https://github.com/prash813/matter-mqtt-demo-client.git).
   
```
cp -ar pemcerts /var/snap/matter-hauto-demo/common/
```





