> You are viewing the calico-docker documentation for release v0.10.0.

# Frequently Asked Questions
This page contains answers to some frequently-asked questions about Calico on Docker.

## Can a guest container have multiple networked IP addresses?
Yes. You can add IP addresses using the `calicoctl container <CONTAINER> ip (add|remove) <IP>` command.

## Why isn't the `-p` flag on `docker run` working as expected?
Simply put, you don't need this flag with Calico and so Calico doesn't support it.

The `-p` flag tells Docker to set up port mapping to connect a port on the Docker host
to a port on your container.  This is useful because with Docker bridge networking, the
container's IP address isn't reachable outside the host.  But with Calico, the 
container's IP address is reachable not only within your cluster, but outside as well
(see later questions for more detail).

If you're used to running your containers like this

    docker run -d -p 8080:80 myhttpserver
  
and then accessing them via `<docker-host-ip>:8080`, you can instead run

    docker run -d --publish-service myhttpserver.mynetwork.calico myhttpserver
  
and then access via `<container-ip>:80`.  No more ephemeral ports, or port conflicts over
which container gets to bind to port 80 (or any other port)!

## Can Calico containers use any IP address within a pool, even subnet network/broadcast addresses?

Yes!  Calico is fully routed, so all IP address within a Calico pool are usable as 
private IP addresses to assign to a workload.  This means addresses commonly 
reserved in a L2 subnet, such as IPv4 addresses ending in .0 or .255, are perfectly 
okay to use.

## How do I get network traffic into and out of my Calico cluster?
The recommended way to get traffic to/from your Calico network is by peering to 
your existing data center L3 routers using BGP and by assigning globally 
routable IPs (public IPs) to containers that need to be accessed from the internet. 
This allows incoming traffic to be routed directly to your containers without the 
need for NAT.  This flat L3 approach delivers exceptional network scalability
and performance.

A common scenario is for your container hosts to be on their own 
isolated layer 2 network, like a rack in your server room or an entire data 
center.  Access to that network is via a router, which also is the default 
router for all the container hosts.

If this describes your infrastructure, [this guide](ExternalConnectivity.md) 
explains in more detail what to do. Otherwise, detailed datacenter networking 
recommendations are given in the main 
[Project Calico documentation](http://docs.projectcalico.org/en/latest/index.html).
We'd also encourage you to [get in touch](http://www.projectcalico.org/contact/) 
to discuss your environment.

### How can I enable NAT for outgoing traffic from containers with private IP addresses?
If you want to allow containers with private IP addresses to be able to access the 
internet then you can use your data center's existing outbound NAT capabilities
(typically provided by the data center's border routers).

Alternatively you can use Calico's built in outbound NAT capability by enabling it on any
Calico IP pool. In this case Calico will perform outbound NAT locally on the compute
node on which each container is hosted.
```
./calicoctl pool add <CIDR> --nat-outgoing
```
Where `<CIDR>` is the CIDR of your IP pool, for example `192.168.0.0/16`.

Remember: the security profile for the container will need to allow traffic to the 
internet as well. You can read about how to configure security profiles in the 
[Advanced Network Policy](AdvancedNetworkPolicy.md) guide.

### How can I enable NAT for incoming traffic to containers with private IP addresses?
As discussed, the recommended way to get traffic to containers that 
need to be accessed from the internet is to give them public IP addresses and
to configure Calico to peer with the data center's existing L3 routers.

In cases where this is not possible then you can configure incoming NAT 
(also known as DNAT) on your data centers existing border routers. Alternatively
you can configure incoming NAT with port mapping on the host on which the container
is running on. 
```
iptables -A PREROUTING -t nat -i eth0 -p tcp --dport <EXPOSED_PORT> -j DNAT  --to <CALICO_IP>:<SERVICE_PORT>
```
For example, you have a container to which you've assigned the CALICO_IP of 192.168.7.4, and you have NGINX running on port 80 inside the container. If you want to expose this service on port 80, then you could run the following command:
```
iptables -A PREROUTING -t nat -i eth0 -p tcp --dport 80 -j DNAT  --to 172.168.7.4:80
```
The command will need to be run each time the host is restarted.

Remember: the security profile for the container will need to allow traffic to the exposed port as well.  You can read about how to configure security profiles in the [Advanced Network Policy](AdvancedNetworkPolicy.md) guide.

### Can I run Calico in a public cloud environment? 
Yes.  If you are running in a public cloud that doesn't allow either L3 peering or L2 connectivity between Calico hosts then you can specify the `--ipip` flag your Calico IP pool:
```
./calicoctl pool add <CIDR> --ipip --nat-outgoing
```
Calico will then route traffic between Calico hosts using IP in IP.

## Orchestrator integration

For a lower level integration see [Orchestrators](Orchestrators.md).

