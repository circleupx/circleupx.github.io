---
title: Kubernetes In WSL - Connect to a service from Windows
tags: [WSL, Kubernetes, Microk8s]
author: "Yunier"
date: "2022-12-14"
description: "How to connect to a Kubernetes service running on WSL from Windows?"
series: [WSL]
---

Today's post is a follow-up to my [Use Kubernetes In WSL](/post/2022/use-kubernetes-in-wsl/) blog post, where I outlined how to install Kubernetes on WSL. As noted at the end of the post, I was having issues connecting from the host, a windows machine, to Kubernetes in WSL.

## Connection Issue

The main issue I was facing was that I could not connect to a pod running on Kubernetes using window's localhost. Take the following Nginx deployment obtained from the [official](https://k8s.io/examples/controllers/nginx-deployment.yaml) Kubernetes documentation.

```YAML
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80

```

I can use the YAML content above to create a [deployment](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#-em-deployment-em-) by executing the following kubectl command.

```bash
kubectl apply -f https://k8s.io/examples/controllers/nginx-deployment.yaml
```

I can confirm the deployment was successful by verifying that all 3 pods are up and running using the following kubectl command.

```bash
kubectl get pods
```

Once the pods are ready to receive traffic we can expose our deployment using the following kubectl command.

```bash
kubectl expose deployment nginx-deployment --port=31080 --target-port=80 --type=NodePort
```

- **target-port** is where the container is listening for requests coming from outside the node.
- **port** is where the container is listening inside the cluster, in this case, port 80.

Once the service has been created you can use the following kubectl command to see the cluster port, the service port assigned to the Nginx deployment service, and the randomly generated port for local connections.

```bash
kubectl get svc -o wide
```

Here is my output of the command above.

```bash
NAME               TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)           AGE     SELECTOR
kubernetes         ClusterIP   10.152.183.1     <none>        443/TCP           9m27s   <none>
nginx-deployment   NodePort    10.152.183.122   <none>        31080:30454/TCP   83s     app=nginx
```

This is where I incorrectly assumed that I could reach the Nginx service running on Kubernetes from Windows' host using localhost. I made this assumption because [WSL's localhost now by default binds to Windows' localhost](https://learn.microsoft.com/en-us/windows/wsl/networking#accessing-linux-networking-apps-from-windows-localhost). I assumed that by exposing the deployment, the randomly generated port, 30454, which is used to connect to the service through localhost on WSL would bind to port 30454 on the Window's host and allow me to access the Nginx service. 

These assumptions were wrong. The following screenshots confirm it.

<div style="padding: 15px; border: 1px solid transparent; border-color: transparent; margin-bottom: 20px; border-radius: 4px; color: #8a6d3b;; background-color: #fcf8e3; border-color: #faebcc;">
Failed to connect to port 30454 from Windows' localhost.
</div>

![Failed to connect on port 30454](/post/2022/connect-to-a-service-in-a-kubernetes-instance-hosted-on-wsl-from-windows/localhost-failed-on-port-30454.png)

I really needed to figure out a way to connect to the Nginx app running in k8s on WSL while using localhost from windows. I ended up chasing four possible solutions that I now want to share with you.

## Using the Node IP 

The first approach I took to connect to Nginx from Windows was to use the node IP. If you followed the commands under [Connection Issue](/post/2022/connect-to-a-service-in-a-kubernetes-instance-hosted-on-wsl-from-windows#connection-issue) and have all three Nginx pods up and running then replace localhost with the node IP on your browser in Windows. 

In my case, the node IP is 172.23.207.235, I obtain that value by running the following command.

```bash
kubectl get node -o wide
```

Here is the output of the command above.

```bash
NAME    STATUS   ROLES    AGE   VERSION   INTERNAL-IP      EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION                      CONTAINER-RUNTIME
gohan   Ready    <none>   30m   v1.25.4   172.23.207.235   <none>        Ubuntu 22.04.1 LTS   5.15.79.1-microsoft-standard-WSL2   containerd://1.6.9
```

To connect to NGINX I opened up my Brave browser to 172.23.207.235:30454.

<div style="padding: 15px; border: 1px solid transparent; border-color: transparent; margin-bottom: 20px; border-radius: 4px; color: #8a6d3b;; background-color: #fcf8e3; border-color: #faebcc;">
The following screenshot shows that I can connect to a service running in Kubernetes on WSL from Windows using the node IP address.
</div>

![Connect to the service using node ip](/post/2022/connect-to-a-service-in-a-kubernetes-instance-hosted-on-wsl-from-windows/connected-to-service-using-node-ip.png)

Bingo, I can reach the service from Windows. If you are not a fan of having to use an IP address then may I suggest taking the [modifying Windows host files](https://www.howtogeek.com/784196/how-to-edit-the-hosts-file-on-windows-10-or-11/) approach. 

In my case, **I wanted to connect to the service using localhost**, so I thought, why not map 172.23.207.235 to Window's localhost, as it turns out, that is big fat **NO**. I learned that you should never attempt it, leave localhost alone. Instead map it to something similar, perhaps local.host or www.localhost.com

<div style="padding: 15px; border: 1px solid transparent; border-color: transparent; margin-bottom: 20px; border-radius: 4px; color: #8a6d3b;; background-color: #fcf8e3; border-color: #faebcc;">
The following screenshot show you can create your own version of localhost.
</div>

![Connected using a customer local host](/post/2022/connect-to-a-service-in-a-kubernetes-instance-hosted-on-wsl-from-windows/connected-to-service-using-custom-loca-host.png)

If you still cannot reach the service after having modified the host's file, flush your DNS in Windows using the following command.

```Powershell
ipconfig /flushdns
```
Then I realize that there were two flaws with this approach. 

1. The port is randomly generated when the service is created.
2. The Node IP can change if you restart or shut down WSL. **As of December 2022, there is no easy way to set the IP of WSL to be static.**

I did not find this approach to be a viable solution so I moved on to the next approach, port forwarding.
 
## Using Port-Forward

This approach does not involve any modifications of host files, and there is no need to obtain the node IP. It involves using [port forwarding](https://kubernetes.io/docs/tasks/access-application-cluster/port-forward-access-application-cluster/)

After following the commands under [Connection Issue](/post/2022/connect-to-a-service-in-a-kubernetes-instance-hosted-on-wsl-from-windows#connection-issue) and having verified that all 3 pods are up and running, use the following kubectl command to port forward from the WSL's localhost to Windows' localhost. Remember, [they are now one and the same](https://learn.microsoft.com/en-us/windows/wsl/networking#accessing-linux-networking-apps-from-windows-localhost).

To start port forwarding traffic to the Nginx pods run the following command.

```bash
kubectl port-forward --address 0.0.0.0 service/nginx-deployment 31080
```

Kubernetes will forward traffic from 31080 to port 80 on the container, and since WSL's localhost:31080 is now the same as Windows port 31080, I can open up the browser to localhost:31080 to connect to the service.

<div style="padding: 15px; border: 1px solid transparent; border-color: transparent; margin-bottom: 20px; border-radius: 4px; color: #8a6d3b;; background-color: #fcf8e3; border-color: #faebcc;">
The following screenshot shows that you can connect using localhost so long as you are port-forwarding the traffic in WSL.
</div>

![Connecte to the service using port-forward](/post/2022/connect-to-a-service-in-a-kubernetes-instance-hosted-on-wsl-from-windows/connected-to-the-service-using-port-forward.png)

Another approach would be to forward traffic from the pod instead of the service using the following command, the result is the same.

```Bash
kubectl port-forward nginx-deployment-7fb96c846b-4c4r4 32196:80
```

- **nginx-deployment-7fb96c846b-4c4r4** is the name of one of the three pods running.

This is great, I love being able to connect using localhost, however, this solution is temporary, as soon as you stop port-forwarding traffic, the connection will stop work working on Windows. 

On to the next approach, using MetalLB.

## Using MetalLB

This approach involves using a microk8s addon, [MetalLB](https://metallb.org/), to allow load balancing. After going through it I realized that this approach is exactly as [Using Node Ip](/post/2022/connect-to-a-service-in-a-kubernetes-instance-hosted-on-wsl-from-windows#node-ip). If you didn't like that solution then you can skip this part or not, you can learn how to use MetalLB. Fun!

MetalLB is a load-balancer implementation for bare metal Kubernetes clusters, using standard routing protocols. It can be enabled in microk8s using the following command.

```bash
microk8s enable metallb
```

Note that when you execute the command, MetalLb is going to expect you to provide an IP. You can specify it as a range like 10.64.140.43-10.64.140.49,192.168.0.105-192.168.0.111 or using CDIR notation. I prefer CDIR notation.

First, I need an IP address. If I run the following command I will get the IP of the node.

```bash
kubectl get node -o wide
```

Gives me the following output.

```bash
NAME    STATUS   ROLES    AGE   VERSION   INTERNAL-IP     EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION                      CONTAINER-RUNTIME
gohan   Ready    <none>   19h   v1.25.4   172.23.200.34   <none>        Ubuntu 22.04.1 LTS   5.15.79.1-microsoft-standard-WSL2   containerd://1.6.9
```

For my load balancer IP, I'm going to change the last octet from 34 to 49, so the IP for MetalLb is going to be 172.23.200.49.

```bash 
microk8s enable metallb:172.23.200.49/32
```

Note that I'm using /32, this keeps the load balancer IP static. If you don't understand why then may I recommend watching [Understanding CIDR Ranges and dividing networks](https://www.youtube.com/watch?v=MmA0-978fSk)

Delete the Nginx service if you created one.

```bash
kubectl delete svc nginx-deployment
```

Now expose the deployment again but this time the type will be LoadBalancer, not NodePort

```bash
kubectl expose deployment nginx-deployment --port=31080 --target-port=80 --type=LoadBalancer
```

Back on Windows, I'll open a web browser and navigate to http://172.23.200.49:31080/ to confirm I can reach the Nginx service.

As expected, I can reach it.

![Nginx load balancer](/post/2022/connect-to-a-service-in-a-kubernetes-instance-hosted-on-wsl-from-windows/nginx-using-load-balancer.png)

Just like the [Using the Node IP ](/post/2022/connect-to-a-service-in-a-kubernetes-instance-hosted-on-wsl-from-windows#node-ip) approach, if don't enjoy using an IP address to access the service, modify the hosts file and map the IP of the load balancer to a custom domain. 

You can could also port proxy the traffic from Windows to WSL using netsh interface portproxy.

For example, 

```Powershell
netsh interface portproxy add v4tov4 listenport=31080 listenaddress=0.0.0.0 connectport=31080 connectaddress=172.23.207.235
```

Remember, if you restart WSL, a new IP will be assigned to the node, which means your load balance IP will no longer router traffic, you will need to reenable MetalLB using the new node IP to get traffic flowing into the Kubernetes service.

## Using HostPort

What ultimately ended up being my preferred solution. HostPort keeps everything simple, no hosts files, no IPs, and no fuss. I consider this approach to be the same as [port forwarding](/post/2022/connect-to-a-service-in-a-kubernetes-instance-hosted-on-wsl-from-windows#port-forwarding) but unlike Port Forwarding, this approach has a permanent solution, well so long as the pods are running.

HostPort applies to the Kubernetes containers. The port is exposed to the WSL network host. This is often an approach not recommended because the Host IP can change. In WSL that happens when WSL is restarted or shut down.

To see it in action start from a clean slate, and delete all nginx-deployments & nginx-services running, now we are going to deploy Nginx again but this time we are going to modify the deployment by adding an additional configuration, hostPort as seen in the YAML below which is the content of my deploy.yaml file. 

The number of replicas was reduced from 3 to 1, because as mentioned above, the host port applies to a container running in a pod, and you cannot map one port to multiple containers. You could leave the number of containers as 3 but note that when the pods are created, Kubernetes will only set one of the three as ready, that one pod that is ready will be the only one that can serve traffic.

```YAML
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
          hostPort: 5700

```

As you can tell from the YAML, I decide to use port 5700. I can now apply this deployment using the following command.

```bash
kubectl apply -f deploy.yaml
```

You can use describe to see that the deployment is now bound to port 5700 in WSL.

```bash
kubectl describe deployment nginx-deployment
```

```bash
Name:                   nginx-deployment
Namespace:              default
CreationTimestamp:      Thu, 15 Dec 2022 23:59:27 -0500
Labels:                 app=nginx
Annotations:            deployment.kubernetes.io/revision: 1
Selector:               app=nginx
Replicas:               1 desired | 1 updated | 1 total | 1 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  25% max unavailable, 25% max surge
Pod Template:
  Labels:  app=nginx
  Containers:
   nginx:
    Image:        nginx:1.14.2
    Port:         80/TCP
    Host Port:    5700/TCP
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      False   MinimumReplicasUnavailable
  Progressing    True    ReplicaSetUpdated
OldReplicaSets:  <none>
NewReplicaSet:   nginx-deployment-66cdbdf488 (1/1 replicas created)
Events:          <none>
```

Great, need one additional configuration. This time on the host, open up Powershell as an administrator and execute the following command.

```Powershell
netsh interface portproxy add v4tov4 listenport=5700 listenaddress=0.0.0.0 connectport=5700 connectaddress=172.23.207.235
```

Where listenport is a port in our window's host, listenaddress is the host's localhost, connectport is the host port defined in our deploy.yaml file, and connectaddress is the node IP address obtained using the following command.

```bash
kubectl get node -o wide
```

After the **netsh interface portproxy** command is executed I was ready to connect. I opened a browser to localhost:5700 on the Windows machine.

<div style="padding: 15px; border: 1px solid transparent; border-color: transparent; margin-bottom: 20px; border-radius: 4px; color: #8a6d3b;; background-color: #fcf8e3; border-color: #faebcc;">
The following screenshot shows that I can connect to the Nginx service from Windows' localhost.
</div>

![Connected to service using Host Port](/post/2022/connect-to-a-service-in-a-kubernetes-instance-hosted-on-wsl-from-windows/connectec-to-service-using-node-port.png)

Perfect I can now connect to Nginx as if it were natively running on Windows. Quick tip, before deciding which port to use on the command netsh interface portproxy run the following command.

```Powershell
netsh interface portproxy show all
```

It will output any mapping, you may already have created or had created by another service. The port listed is unavailable, and therefore, cannot be remapped unless you delete it using the following command.

```bash
netsh interface portproxy delete v4tov4 listenport=5700 listenaddress=0.0.0.0
```

Using the host port solves my original issue, I can now connect to services running on Kubernetes in WSL from Windows.

## Conclusion

For now, I feel like HostPort is the best solution I could come up with. If the day ever comes when I can set a static IP in WSL then MetalLB would probably be my preferred choice since HostPort limits the number of PODs to one.


Thanks for reading.