# Docker Swarm, Traefik and Portainer on the GCP
On the GCP, reproducing the Traefik and the Portainer setup on 3 Node Docker Swarm.

## Create the VM instance
In the Google Cloud console, create and start 3 VMs and add 3 persistent disks for using data sharing.

## Common Initialize
```
$ sudo apt-get update; sudo apt-get install -y vim tzdata apt-transport-https ca-certificates curl gnupg lsb-release
$ sudo ip link
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: ens4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1460  # <--
    ...
$ sudo vi /etc/netplan/50-cloud-init.yaml
...
network:
    ethernets:
        ens4:
            dhcp4: true
            mtu: 1500   # <-- set to 1500
...
$ sudo reboot
```

## GlusterFS for shared data
Initialize added disk and install the GlusterFS on all nodes:

### Initialize disk
```
$ sudo lsblk
NAME    MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
...
sda       8:0    0    10G  0 disk
├─sda1    8:1    0   9.9G  0 part /
├─sda14   8:14   0     4M  0 part
└─sda15   8:15   0   106M  0 part /boot/efi
sdb       8:16   0   100G  0 disk           # <--

$ sudo mkfs.ext4 -m 0 -E lazy_itable_init=0,lazy_journal_init=0,discard /dev/sdb

$ sudo mkdir -p /gluster_disk
$ sudo mount -o discard,defaults /dev/sdb /gluster_disk
$ sudo chmod a+w /gluster_disk

$ sudo blkid /dev/sdb                       # check disk UUID

$ sudo vi /etc/fstab
LABEL=.....
UUID=360ad7e0-b4c6-4fc0-822d-94fcf7c870f7 /gluster_disk ext4 discard,defaults,nofail 0 2
```

### Install the GlusterFS
```
$ sudo vi /etc/hosts
...
10.178.0.1 node1
10.178.0.2 node2
10.178.0.3 node3
...

$ sudo apt list --installed glust*          # check glusterfs
$ sudo apt-get install -y glusterfs-server
$ sudo systemctl start glusterd
$ sudo systemctl enable glusterd
$ sudo systemctl status glusterd
```
```
$ sudo gluster peer probe node1
peer probe: success
$ sudo gluster peer probe node2
peer probe: success
$ sudo gluster peer probe node3
peer probe: success
$ sudo gluster peer status
Number of Peers: 2
```
```
$ sudo gluster volume create gluster_vol1 replica 3 transport tcp node1:/gluster_disk/vol1 node2:/gluster_disk/vol1 node3:/gluster_disk/vol1
volume create: gluster_vol1: success: please start the volume to access data

$ sudo gluster volume start gluster_vol1
volume start: gluster_vol1: success

$ sudo gluster volume info gluster_vol1
Volume Name: gluster_vol1
Type: Replicate
Volume ID: 06.....7-6f97-4754-ab30-ca......9d16
Status: Created
Snapshot Count: 0
Number of Bricks: 1 x 3 = 3
Transport-type: tcp
Bricks:
Brick1: node1:/gluster_disk/vol1
Brick2: node2:/gluster_disk/vol1
Brick3: node3:/gluster_disk/vol1
Options Reconfigured:
cluster.granular-entry-heal: on
storage.fips-mode-rchecksum: on
transport.address-family: inet
nfs.disable: on
performance.client-io-threads: off
```
```
$ sudo mkdir /gluster_data
$ sudo mount -t glusterfs node1:/gluster_vol1 /gluster_data
$ sudo vi /etc/fstab
node1:/gluster_vol1            /gluster_data      glusterfs     _netdev,auto,x-systemd.requires=glusterd.service 0 0
```

## Install Docker
Install Docker on all nodes:
```
$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
$ echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
$ sudo apt-get update; sudo apt-get install -y docker-ce docker-ce-cli containerd.io
```
```
$ sudo curl -L "https://github.com/docker/compose/releases/download/v2.23.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
$ sudo chmod +x /usr/local/bin/docker-compose
$ sudo groupadd docker; sudo gpasswd -a ${USER} docker
```

## Initialize the Swarm
Initialize Swarm on Manager(node-1)
```
$ sudo docker swarm init --advertise-addr 10.178.0.1
==> sudo docker swarm join --token SWMTKN-1-2cgy6aua1lb3b5ncxjcm9yx91oy01tsi6dmunf5wwicr1wk2ur-cvlskbiwt2yaqpsbi6cy5gucu 10.178.0.1:2377
```
Join Worker Node to the Swarm(node-2, node-3)
```
$ sudo docker swarm join --token SWMTKN-1-2cgy6aua1lb3b5ncxjcm9yx91oy01tsi6dmunf5wwicr1wk2ur-cvlskbiwt2yaqpsbi6cy5gucu 10.178.0.1:2377
==> This node joined a swarm as a worker.
```
Lists nodes from the Manager(node-1)
```
$ sudo docker node ls
ID                            HOSTNAME     STATUS    AVAILABILITY   MANAGER STATUS   ENGINE VERSION
wetzajts0qrlu6wp1kul3hhvi *   instance-1   Ready     Active         Leader           ...
8qi29fuqqqunha65mtnrmh2cf     instance-2   Ready     Active         Reachable        ...
ve5iqxqxlkfejqci1ptatnnfs     instance-3   Ready     Active         Reachable        ...
```

## Initialize Docker Network
Initialize networks on Manager(node-1)
```
$ sudo docker network create --attachable --driver overlay backend
$ sudo docker network create --attachable --driver overlay frontend
$ sudo docker network create --attachable --driver overlay portainer_network

$ sudo docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
...
u43m8mxybuom        backend             overlay             swarm
snk02wjdva5n        frontend            overlay             swarm
ksl9zd0feun7        portainer_network   overlay             swarm
```

## Traefik
Create the compose file for traefik `docker-compose-traefik.yml`:
```
version: "3.7"

networks:
  frontend:
    external: true

services:
  traefik:
    image: traefik:v2.10
    command:
      - --api=true
      - --api.insecure=true # set to 'false' on production
      - --api.dashboard=true # see https://docs.traefik.io/v2.0/operations/dashboard/#secure-mode for how to secure the dashboard
#      - --api.debug=true # enable additional endpoints for debugging and profiling
#      - --log.level=DEBUG # debug while we get it working, for more levels/info see https://docs.traefik.io/observability/logs/
      - --providers.docker=true
      - --providers.docker.swarmMode=true
      - --providers.docker.exposedbydefault=false
      - --providers.docker.network=frontend
      - --entrypoints.web.address=:80
      - --entryPoints.web.forwardedHeaders.trustedIPs=10.0.0.0/16
      - --entryPoints.web.forwardedHeaders.insecure
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    networks:
      - frontend
    ports:
      - "80:80"
      - "8080:8080"
    deploy:
      placement:
        constraints:
          - node.role == manager
```
```
$ sudo docker stack deploy -c ./docker-compose-traefik.yml traefik
$ sudo docker stack ps traefik
```

## Portainer with agent
### Portainer agent
Create the compose file for portainer agent `docker-compose-portagent.yml`:
```
version: "3.7"

networks:
  portainer_network:
    external: true

services:
  portainer_agent:
    image: portainer/agent:alpine
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - /var/lib/docker/volumes:/var/lib/docker/volumes
    networks:
      - portainer_network
    environment:
      - "AGENT_CLUSTER_ADDR=tasks.portainer_agent"
    deploy:
      mode: global
```
```
$ sudo docker stack deploy -c ./docker-compose-portagent.yml portagent
$ sudo docker stack ps portagent
```
### Portainer
Create the compose file for portainer `docker-compose-portainer.yml`:
```
version: "3.7"

networks:
  frontend:
    external: true
  portainer_network:
    external: true

services:
  portainer:
    image: portainer/portainer-ce:alpine
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - /gluster_data/portainer_data:/data
    networks:
      - frontend
      - portainer_network
    deploy:
      placement:
        constraints:
          - node.role == manager
      replicas: 1
      labels:
        - "traefik.enable=true"
        - "traefik.http.services.portainer.loadbalancer.server.port=9000"
        - "traefik.http.routers.portainer.rule=Host(`some.domain.xyz`)"
        - "traefik.http.routers.portainer.service=portainer"
        - "traefik.http.routers.portainer.entrypoints=web"
        # - "traefik.http.routers.portainer.middlewares=portainer-whitelist"
        # - "traefik.http.middlewares.portainer-whitelist.ipwhitelist.sourcerange=..."
        # - "traefik.http.middlewares.portainer-whitelist.ipwhitelist.ipstrategy.excludedips=...."
```
```
$ sudo mkdir -p /gluster_data/portainer_data
$ sudo docker stack deploy -c ./docker-compose-portainer.yml portainer
$ sudo docker stack ps portainer
```













