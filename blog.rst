# Getting started with coreos on openstack (using HEAT)

### What is CoreOS and Why?
CoreOS is an open-source lightweight operating system based on the Linux kernel and designed for providing infrastructure to clustered deployments
Microservices architecture have their advantages and coreOS is the operating system to host containerized microservices.

This blog post is experience while learning basics of coreos on openstack. Simple archticture of this blog is:

<<<< image >>>>>

### Prequirisites for the blog
To follow this tutorial, we need to have some binaries on the host machine. 

#### etcdctl
`etcdctl` is a command line client for etcd. CoreOS's etcd is a distributed, consistent key-value store for shared configuration and service discovery.

As per the above architecture, our control node will have etcd service running. But on local machine we need this client to talk to our cluster.
We can install this in one step.

```sh
$ curl -L  https://github.com/coreos/etcd/releases/download/v2.1.1/etcd-v2.1.1-linux-amd64.tar.gz -o /tmp/etcd-v2.1.1-linux-amd64.tar.gz
$ tar xzvf /tmp/etcd-v2.1.1-linux-amd64.tar.gz -C /tmp/
$ mv /tmp/etcd-v2.1.1-linux-adm64/etcdctl /usr/local/bin
```
> Adds etcdctl into system's PATH for binary to be found.

#### fleetctl
fleet ties together systemd and etcd into a simple distributed init system. Think of it as an extension of systemd that operates at the cluster level instead of the machine level. 

fleet provides a command-line tool called fleetctl. We will use this to talk to our cluster

```sh
$ curl -L https://github.com/coreos/fleet/releases/download/v0.11.2/fleet-v0.11.2-linux-amd64.tar.gz -o fleet-v0.11.2-linux-amd64
$ tar xzvf /tmp/etcd-v2.1.1-linux-amd64.tar.gz -C /tmp/
$ mv /tmp/etcd-v2.1.1-linux-adm64/fleetctl /usr/local/bin
```

#### python-heatclient & python-glanceclient
Openstack's heat client is required to spinup VM's in openstack infrastructure.
```sh
$ pip install python-heatclient python-glanceclient
```
##### conclusion
Lets verify if all the binaries are setup correctly.

Each of these `fleetctl --version` , `etcdctl --version`, `heat --version` & `glance --version` should return correct oputpu.

## Start a single node cluster

Lets start with keeping things simple and spinning up a single node cluster (only control node). This will start a single coreos node with fleet and etcd running on it.

```sh
$ heat stack-create -f heat-template.control.yaml -P discovery_token_url=`curl -q https://discovery.etcd.io/new?size=1` trycoreos

+--------------------------------------+---------------------+-----------------+----------------------+
| id                                   | stack_name          | stack_status    | creation_time        |
+--------------------------------------+---------------------+-----------------+----------------------+

| 897f08ad-4beb-4000-a871-aaa0231ade90 | trycoreos_alok_0341 | CREATE_COMPLETE | 2015-07-30T22:11:35Z |
+--------------------------------------+---------------------+-----------------+----------------------+

```
> If this command returns succesfully means our machine is being provisioned. We can check status of the machine with `heat stack-show trycoreos`. Once success ,we can get the floating point ip with `heat output-show trycoreos control_ip`.

Now that we have our node ready, we can ssh to it directly but lets try fleetctl to see status of our cluster.
```
$  FLEETCTL_TUNNEL="180.148.26.211" fleetctl list-machines
MACHINE                           IP                               METADATA
0315e138...                 192.168.222.2                      role=control
```
> Lets list machines of our cluster with fleet.

Fleetctl can be used to monitor and start/stop different services on our cluster. We will try to cover this in a separate topic.

```sh
$ fleetctl ssh 0315e138
``` 
> logs us into control node

`ETCDCTL_PEERS="https://<ip_address>:2379" etcdctl ls`

`etcd ls --recursive`
`etcd set topic coreos'
`etcd get topic`
##### etcd on http
`curl -X GET http://180.142.33.2:2379/version`

`curl -X GET http://180.142.33.2:2379/v2/keys`

#### Insight
Lets get deepeer and see how this works. As part of heat-template.control.yaml, we are provisioning a single node with cloud config:
```
#cloud-config
coreos:
    fleet:
        etcd_servers: http://127.0.0.1:2379
        metadata: role=control
    etcd2:
        name: etcd2
        discovery: $token_url
        advertise-client-urls: http://$public_ipv4:2379
        initial-advertise-peer-urls: http://$public_ipv4:2380
        listen-client-urls: http://0.0.0.0:2379
        listen-peer-urls: http://0.0.0.0:2380
    units:
        - name: etcd2.service
          command: start
        - name: fleet.service
          command: start
    update:
        group: alpha
        reboot-strategy: reboot
```
This follows starndard coreos cloud init guide to initialize a system. fleet and etcd2 services are already existing on coreos alpha channel. We try to override the default etcd2 and fleet configuration with custom parameters. 
What is discovery token.
These can be seen in
`$ cat /run/systemd/system/etcd2.service.d/20-cloud-init`
Similarly 
`$ cat /run/systemd/system/fleet.service.d/20-cloud-init`

We can verify status of both the services after sshing into node
`systemctl status etcd2`
`systemctl status fleet`
### Explanation:

    Lets set/get some keys
    `etcdctl set topic coreos`
    `etcdctl get topic`
    This key value pair is shared across different node. For now it is only 
    `fleetctl list-machines`
    We can verify there is only one machine with control role. Roles define.. .
    `fleetctl list-units`
    # What is fleetctl. This is another type of init system which works across machines.
    this works with a tunnel FLEETCTL_TUNNEL.

## Start a multi node cluster

## Dynamically add nodes to a cluster
Add new machines to cluster.    
## How this works
cloud-init.
shell script provisioner to put cloud-init into place and 
update discovery token. This is needed because.

# Step 2 Master slave:
`git checkout -- .`
`vagrant destroy -f`
`vagrant up`
# Disclaimer: 
this is inspired by talks from.
