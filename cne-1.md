# Welcome to the Cloud Network Engineer Discovery Lab!
This short lab will introduce you to the features presented in the Cloud Network Engineer journey of the Isovalent Cilium labs.
In particular, you will learn about:

BGP with Cilium
IPv6 networking
Load-Balancer Service IP Address Management (LB-IPAM)
BGP Service Announcement
L2 Service Announcement
Egress Gateway

# 🌐 IPv6 on Kubernetes
Kubernetes is not only IPv6-ready but it also provides a transitional pathway from IPv4 to IPv6.

With Dual Stack, each pod is allocated both an IPv4 and an IPv6 address, so it can communicate both with IPv6 systems and the legacy apps and cloud services that use IPv4.

In order to run Dual Stack on Kubernetes, you need a CNI that supports it: of course, Cilium does.

In this lab, you will be using a Dual Stack IPv4/IPv6 cluster and learn not just how Cilium can support dual stack but also advertise both routes over BGP.

Ready? Let's go!

# 🏝️ Connecting the Kubernetes Island
From a networking perspective, a Kubernetes cluster is akin to an island.

Within the island, there are roads connecting all the Pods and they can communicate freely.

To connect the rest of the island (our cluster) to the mainland (our data center), we are going to need a bridge (BGP).

In this lab, you will get a taster of how Cilium natively supports BGP and how you can use it to connect your cluster to your DC network!

# 🏅 Get a Badge!
By completing this lab, you will be able to earn a badge.
Make sure to finish the lab in order to get your badge!


# ✅ Check Cilium Status
In this lab, a Kind Kubernetes cluster has been deployed, with Cilium to provide various network functions. The cluster was deployed in Dual Stack IPv4/IPv6 mode.

In order to verify that Cilium is properly installed and functioning, enter the following in the >_ Terminal tab:

cilium status --wait
Cilium and Operator should be marked OK while some features will be marked as disabled (that's expected).

Now that we know that the Kubernetes cluster is ready, let's pretend for the duration of this lab that your cluster is located on the other side of the galaxy 💫, on a planet called Batuu 🪐, in the Trilon sector.

Your task, as an Imperial officer specialized in inter-galactic telecommunications, is to set up communications between the remote outpost on Batuu and the central base.

Note
Yes, the Cilium folks tend to overdo the Star Wars references. We hope you are familiar with Star Wars. If not, don't worry, you don't need to know any of the Star Wars trivia to enjoy this lab.


# 🛣️ Overlay vs Direct Routing
Cilium allows to configure Kubernetes clusters either using an overlay network (VXLAN of Geneve) or direct routing.

This lab is configured with direct routing, and the pod CIDRs are propagated using BGP, taking advantage of an existing BGP infrastructure.

In this lab, our remote BGP peer is using FRR (Free Range Routing) and a virtual networking platform called containerlab.

Switch to the 🔗 🗺️ Network Topology tab to observe the various pieces of the infrastructure:

router0 is the Top of Rack router
srv-control-plane, srv-worker, and srv-worker2 are the Kubernetes nodes
srv-client is a separate node, which hosts an Imperial outpost

# 🤝 BGP Peering
Cilium natively supports BGP and enables you to set up BGP peering with network devices such as Top of Rack devices and advertise Kubernetes IP ranges (Pod CIDRs and Service IPs) to the broader data center network.

Cilium is configured to peer with the BGP Top of Rack router using CiliumBGPPeeringPolicy resources. Three of them have been deployed, one per node.

In the >_ Terminal tab, list them with:

kubectl get ciliumbgppeeringpolicy
Inspect the policy for the Control Plane node with:

kubectl get ciliumbgppeeringpolicy control-plane -o yaml | yq '.spec'
The key aspects of the policy are:

the remote peer IP address (peerAddress) and AS Number (peerASN)
your own local AS Number (localASN)
And that's it!

In this lab, we peer with the IPv6 address of the Top of Rack router (fd00:10:0:1::1).

The Autonomous System (AS) number for the control-plane node is 65001 and our remote peer's ASN is 65000: the BGP session will be an eBGP (external BGP) session as our AS numbers are different.

🚀 Verify successful BGP peering
With the BGP peering set, the peering sessions between the Cilium nodes and the Top of Rack routers should be established successfully.

Let's verify that the sessions have been established and that routes are learned successfully (it might take a few seconds for the sessions to come up).

Run this command again:

cilium bgp peers
Expect this output:

```console
Node                 Local AS   Peer AS   Peer Address     Session State   Uptime   Family         Received   Advertised
kind-control-plane   65001      65000     fd00:10:0:1::1   established     2m48s    ipv4/unicast   3          1
                                                                                    ipv6/unicast   3          1
kind-worker          65002      65000     fd00:10:0:2::1   established     2m48s    ipv4/unicast   3          1
                                                                                    ipv6/unicast   3          1
kind-worker2         65003      65000     fd00:10:0:3::1   established     2m48s    ipv4/unicast   3          1
                                                                                    ipv6/unicast   3          1
```

The BGP sessions have been established!

Log on to the central base's router clab-bgp-cplane-dev-dual-router0 and check if the central base has received the routes over BGP:

```console
docker exec -it clab-bgp-cplane-dev-dual-router0 \
  vtysh -c 'show bgp ipv6 '
The output should be:

BGP table version is 3, local router ID is 10.0.0.1, vrf id 0
Default local pref 100, local AS 65000
Status codes:  s suppressed, d damped, h history, * valid, > best, = multipath,
               i internal, r RIB-failure, S Stale, R Removed
Nexthop codes: @NNN nexthop's vrf id, < announce-nh-self
Origin codes:  i - IGP, e - EGP, ? - incomplete
RPKI validation codes: V valid, I invalid, N Not found

   Network          Next Hop            Metric LocPrf Weight Path
*> fd00:10:1::/64   fd00:10:0:1::2                         0 65001 i
*> fd00:10:1:1::/64 fd00:10:0:3::2                         0 65003 i
*> fd00:10:1:2::/64 fd00:10:0:2::2                         0 65002 i

Displayed  3 routes and 3 total paths
```

ℹ️ Diving Deeper

A detailed explanation of how containerlab works with Cilium can be found in the 🧪 BGP on Cilium lab.


# 6️⃣ Dual Stack Support
Given the size of the Empire and the millions of ships under imperial control, all communications have to be executed over IPv6. As some of the ships only support IPv4, the cluster was deployed in Dual Stack IPv4/IPv6 mode.

Verify that both IPv4 and IPv6 have been activated in the cluster:

cilium config view | grep -i 'enable-ipv. '

🌌 A Star Wars Demo App
The remote outpost on Batuu includes a new Deathstar (the previous one has had an unfortunate accident 💥).

The batuu.yaml manifest will deploy a Star Wars-inspired demo application which consists of:

a batuu Namespace, containing
a deathstar Deployment with 2 replicas
a Kubernetes Service to access the Death Star pods using either IPv4 or IPv6
a tiefighter Deployment with 1 replica
a tiefighter-4 Deployment with 1 replica, using IPv4 to access the Death Star
an xwing Deployment with 1 replica
Deploy the manifest with:

kubectl apply -f batuu.yaml
Wait for the Death Star to be deployed:

kubectl -n batuu rollout status deployment deathstar
Then check the deployments with:

kubectl get -f batuu.yaml
Execute the command a few times until the deployment is marked as fully ready (e.g. 2/2).

Verify that the Deathstar Pods have picked up both an IPv4 and IPv6 address:

kubectl -n batuu describe pods -l class=deathstar | grep -A 2 IPs
Expect to see an ouput like:
```console
IPs:
  IP:           10.1.2.147
  IP:           fd00:10:1:2::7af1
--
IPs:
  IP:           10.1.1.232
  IP:           fd00:10:1:1::94d0
```
The pods have received both an IPv4 and an IPv6 address.

Let's get one of these IPs:

DS1_IP6=$(kubectl -n batuu get po -l class=deathstar -o jsonpath='{.items[0].status.podIPs[1].ip}')
echo $DS1_IP6
Test that you can access one of the Death Star pods via their IP address from the srv-client container in the BGP network:

docker exec -ti clab-bgp-cplane-dev-dual-srv-client curl http://[$DS1_IP6]/v1/
It works, thanks to BGP propagating the container routes to the external network!


ℹ️ Diving Deeper

If you'd like to learn more about IPv6 on Kubernetes, head over to the 🧪 IPv6 Networking and Observability lab.


# 🛰️ Visualizing traffic
Switch to the 🔗 🛰️ Hubble UI tab, a project that is part of the Cilium realm, which lets you visualize traffic in a Kubernetes cluster as a service map.

At the moment, you are seeing four pod identities in the batuu namespace:

xwing is for pods from the xwing deployment
tiefighter represents pods from the tiefighter deployment
tiefighter-4 represents pods from the tiefighter-4 deployment
deathstar represents the various pods deployed by the deathstar deployment
Service Map

The tiefighter, tiefighter-4, and xwing pods make requests to:

the deathstar service (HTTP)
an outpost outside of the cluster (with IP address 10.0.4.2)
In the >_ Terminal tab, observe traffic from the tiefighter to the deathstar, displaying the IP addresses with:

hubble observe \
  --from-pod batuu/tiefighter \
  --to-pod batuu/deathstar \
  --ip-translation=false
Then repeat for the tiefighter-4 identity:

hubble observe \
  --from-pod batuu/tiefighter-4 \
  --to-pod batuu/deathstar \
  --ip-translation=false
As you can see, with the Hubble CLI or UI, you can observe network flows in your cluster, whether they are IPv4 or IPv6!


ℹ️ Diving Deeper

Learn how to observe and debug applications using Hubble and Grafana (and earn a badge!) by taking the 🧪 Golden Signals lab.

How about accessing the Death Star from outside the cluster? We'll explore the options in the next challenge!
Exit
Check


Terminal


🗺️ Network Topology

🛰️ Hubble UI

Editor

💬 Feedback
