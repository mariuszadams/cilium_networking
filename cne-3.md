# ↗️ Egress Gateway
In many Enterprise environments, the applications hosted on Kubernetes need to communicate with workloads living outside the Kubernetes cluster, which are subject to connectivity constraints and security enforcement. Because of the nature of these networks, traditional firewalling usually relies on static IP addresses (or at least IP ranges). This can make it difficult to integrate a Kubernetes cluster, which has a varying —and at times dynamic— number of nodes into such a network.

Cilium’s Egress Gateway feature changes this, by allowing you to specify which nodes should be used by a pod in order to reach the outside world. Traffic from these Pods will be Source NATed to the IP address of the node and will reach the external firewall with a predictable IP, enabling the firewall to enforce the right policy on the pod.


# 🏣 A remote Outpost
The Empire has a remote outpost hosted in the remote-outpost Docker container.

This container is connected to the BGP peering network, and can be accessed directly from the pods.

For security reasons, the outpost's security team wants to identify incoming traffic, and they only allow traffic coming from the 10.0.3.42 IP address.

Try to access it from the tiefighter pod:

kubectl -n batuu exec -ti deployments/tiefighter -- curl http://10.0.4.2:8000
The pod accesses the IP address from its own IPv4 (10.1.1.177), and is denied access.

The same can be seen from the xwing pod (with the 10.1.1.234 IP):

kubectl -n batuu exec -ti deployments/xwing -- curl http://10.0.4.2:8000

# 📡 A Communication from the Outpost
The outpost's security team is contacting you. Answer the call with:

starcom --interactive

# ↗️ Egress Gateway
The kind-worker2 node in the cluster has been configured with a label egress-gw=true to distinguish it from other nodes:

kubectl get no kind-worker2 --show-labels
It has also been assigned the 10.0.3.42 IP address on its net1 interface:

docker exec kind-worker2 ip a show net1
Let's deploy a Cilium Egress Gateway policy that targets pod labeled org=empire. When these pods try to reach the 10.0.4.0/24 (which includes 10.0.4.2) network, the traffic will leave the Kubernetes cluster through a node labeled egress-gw=true, masquerading the source IP from the net1 interface.

Review the policy:

yq egress-gw-policy.yaml
Then apply it:

kubectl apply -f egress-gw-policy.yaml
Now try to access the service from the tiefighter pod:

kubectl -n batuu exec -ti deployments/tiefighter -- curl http://10.0.4.2:8000
It is authorized as the traffic is now masqueraded with the 10.0.3.42 IP address.

Check that the xwing (an alliance ship) is still denied access:

kubectl -n batuu exec -ti deployments/xwing -- curl http://10.0.4.2:8000

ℹ️ Diving Deeper

Learn more! Discover how (and earn a badge!) by taking the 🧪 Cilium Egress Gateway lab.


🏆 You've done it!
You have foiled the plans from the Rebels and played your part in making the Empire more powerful than ever.

Even more importantly, you have discovered, in this short lab, several key networking features of Cilium:

Cilium natively supports IPv6.
The BGP feature can advertise Services to remote hosts and to the entire network.
The Load-Balancer IPAM feature can allocate IP addresses to your Kubernetes services.
The L2 Service Announcements feature can advertise Services locally.
Egress Gateway policies can be used to SNAT traffic exiting the cluster.
Thank you for participating in this lab. Click Check to finish the lab and collect your badge!
Exit
Check


Terminal


Editor

💬 Feedback


