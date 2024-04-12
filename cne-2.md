# LoadBalancer IP Address Management (LB-IPAM)
LoadBalancer IP Address Management (LB-IPAM) is a new elegant feature that lets Cilium provision IP addresses for Kubernetes LoadBalancer Services.

To allocate IP addresses for Kubernetes Services that are exposed outside of a cluster, you need a resource of the type LoadBalancer. When you use Kubernetes on a cloud provider, these resources are automatically managed for you, and their IP and/or DNS are automatically allocated. However, if you run on a bare-metal cluster, in the past, you would have needed another tool like MetalLB to allocate that address.

But maintaining yet another networking tool can be cumbersome and in Cilium 1.13, this is no longer needed: Cilium can allocate IP Addresses to Kubernetes LoadBalancer Services.

# 📣 L2 IP Announcements
Since version 1.13, Cilium has provided a way to create North-South Load Balancer services in the cluster and announce them to the underlying networking using BGP.

However, not everyone with an on-premise Kubernetes cluster has a BGP-compatible infrastructure.

For this reason, Cilium now allows to use ARP in order to announce service IP addresses on Layer 2.

------

# 🌌 Extending the Empire Reach
The Deathstar needs to be accessed over HTTP from the outpost station.

In order for this Imperial base to access the Death Star, you will need to:

Create a Kubernetes Service of the type LoadBalancer to expose your application
Assign an IP address to the Service
Advertise the Service over BGP

# 🪪 Load-Balancer IP Address Management
There are several ways you can expose Kubernetes applications outside of your cluster. One common method is to use Kubernetes Services of the type LoadBalancer.

Check the Death Star service:

kubectl -n batuu get svc deathstar --show-labels
It is already set as a service of type LoadBalancer, but it doesn't have an external IP. The allocation of this IP address is typically done by a cloud provider or by a tool such as MetalLB. Cilium now natively supports this feature, with the use of the Load Balancer IP Address Management (LB-IPAM) feature.

To allocate an IP address, you will need to configure a Cilium LB IP Pool, using the CiliumLoadBalancerIPPool CRD. Inspect the provided manifest:

yq lb-pool.yaml
As you can see, this IP Pool applies to services with an org label set to empire (which the deathstar service has), and it includes both an IPv4 and an IPv6 range.

Deploy the pool with the following command:

kubectl apply -f lb-pool.yaml
Verify it has been deployed successfully with the following command:

kubectl get ciliumloadbalancerippools.cilium.io empire-ip-pool
Verify that the Service has received both an IPv4 and an IPv6 external IPs:

kubectl -n batuu get svc deathstar
Expect to see output similar to this one:

NAME        TYPE           CLUSTER-IP    EXTERNAL-IP                              PORT(S)        AGE
deathstar   LoadBalancer   10.2.15.160   172.18.255.201,2001:db8:dead:beef::111   80:30064/TCP   14m

ℹ️ Diving Deeper

To learn more about IP address service assignment, head over to the 🧪 LoadBalancer IPAM and BGP Service Advertisement lab.


# 📣 Announcing the Service
Retrieve the external IPs for the service:

SERVICE_IP4=$(kubectl -n batuu get svc deathstar -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo $SERVICE_IP4

SERVICE_IP6=$(kubectl -n batuu get svc deathstar -o jsonpath='{.status.loadBalancer.ingress[1].ip}')
echo $SERVICE_IP6
Try to access the service from the srv-client container:

docker exec -ti clab-bgp-cplane-dev-dual-srv-client \
  curl -s --max-time 2 http://[$SERVICE_IP6]/v1/
The request times out, because the service IP is not propagated to the BGP network.

Check the BGP Peering Policies, e.g.:

kubectl get ciliumbgppeeringpolicies.cilium.io control-plane -o yaml | yq '.spec.virtualRouters'
Notice the bottom section:

serviceSelector:
  matchLabels:
    announced: bgp
This means service IPs are announced by this policy, but only for services whose announced has a value of bgp.

Let's patch the deathstar service with this label:

kubectl -n batuu label service/deathstar announced=bgp
Now retry to access the service:

docker exec -ti clab-bgp-cplane-dev-dual-srv-client \
  curl -s --max-time 2 http://[$SERVICE_IP6]/v1/
It works!


ℹ️ Diving Deeper

To learn more about BGP service announcements, head over to the 🧪 LoadBalancer IPAM and BGP Service Advertisement lab.


# 🏛L2 Announcements
As an officer, you also want to be able to check the Death Star service from the host terminal. We'll use IPv4 for this request.

Try it:

curl -s --max-time 2 http://$SERVICE_IP4/v1/
It doesn't work, because the host is not part of the BGP peering network!

However, the host happens to be in the same L2 network as the rest of the containers, so let's use the L2 Load Balancer announcement!

Apply the provided CiliumL2AnnouncementPolicy manifest:

kubectl apply -f layer2-policy.yaml
Then try accessing the service again:

curl -s --max-time 2 http://$SERVICE_IP4/v1/
Cilium is now announcing the service IP via ARP, so the host learned how to route it!


ℹ️ Diving Deeper

Learn more! Discover how (and earn a badge!) by taking the 🧪 L2 Service Announcement lab.

In the next challenge, we'll explore outgoing traffic!
Exit
Check


Terminal


Network Topology

💬 Feedback

