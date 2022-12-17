---
title: Kubernetes In WSL - Connect to a running service from Windows
tags: [WSL, Kubernetes, Microk8s]
author: "Yunier"
date: "2022-12-14"
description: "How to connect to a Kubernetes service running on WSL from Windows?"
series: [WSL]
---

Today's post is a follow-up to my [Use Kubernetes In WSL](/post/2022/use-kubernetes-in-wsl/) blog post, where I outlined how to install Kubernetes on WSL. As noted at the end of the post, I was having issues connecting from the host, a windows machine, to Kubernetes in WSL.

## Connection Issue

The main issue I was facing was that I could not connect to a pod running on Kubernetes using window's localhost. Take the following Nginx deployment to obtain from the [official](https://k8s.io/examples/controllers/nginx-deployment.yaml) Kubernetes documentation.

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

You can confirm the deployment was successful by verifying you have 3 pods up and running using the following kubectl command.

```bash
kubectl get pods
```

Once all three are up and running we can now expose our deployment using the following command.

```bash
kubectl expose deployment nginx-deployment --port=31080 --target-port=80 --type=NodePort
```

Where --target-port is where the container is listening for incoming traffic. The parameter --port is where the service is listening inside the cluster, in this case, port 80.

Once the service has been created you can use the following command to see the cluster port, the service port assigned to the Nginx deployment service, and the randomly generated port for local connections.

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
Failed to connect to port 31080 from Windows' localhost
</div>

![Failed to connnect on port 31080](/post/2022/connect-to-a-service-in-a-kubernetes-instance-hosted-on-wsl-from-windows/localhost-failed-on-port-31080.png)

<div style="padding: 15px; border: 1px solid transparent; border-color: transparent; margin-bottom: 20px; border-radius: 4px; color: #8a6d3b;; background-color: #fcf8e3; border-color: #faebcc;">
Failed to connect to port 30454 from Windows' localhost
</div>

![Failed to connect on port 30454](/post/2022/connect-to-a-service-in-a-kubernetes-instance-hosted-on-wsl-from-windows/localhost-failed-on-port-30454.png)

I really needed to figure out a way to connect to the Nginx app running in k8s on WSL while using localhost from windows. I ended up chasing four possible solutions that I now want to share with you.

## Node IP 

The first approach I took to connect to Nginx from Windows was to use the node IP. If you followed the commands under [Connection Issue](/post/2022connect-to-a-service-in-a-kubernetes-instance-hosted-on-wsl-from-windows#connection-issue) and have all three Nginx pods up and running then replace localhost with the node IP on your browser in Windows. 

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

Bingo, I can reach the service from Windows. If you are not a fan of having an IP address on Browser bars. You could take the [modifying Windows host files](https://www.howtogeek.com/784196/how-to-edit-the-hosts-file-on-windows-10-or-11/) approach. 

In my case, **I wanted to connect to the service using localhost**, so I thought, why not map 172.23.207.235 to Window's localhost, as it turns out, that is big fat **NO**. 

<div style="width:100%;height:0;padding-bottom:57%;position:relative;"><iframe src="https://giphy.com/embed/15aGGXfSlat2dP6ohs" width="100%" height="100%" style="position:absolute" frameBorder="0" class="giphy-embed" allowFullScreen></iframe></div>

I learned that you should never attempt it, trust me, I went down that road, and it doesn't end well, leave localhost alone. Instead map it to something similar, perhaps local.host or www.localhost.com

<div style="padding: 15px; border: 1px solid transparent; border-color: transparent; margin-bottom: 20px; border-radius: 4px; color: #8a6d3b;; background-color: #fcf8e3; border-color: #faebcc;">
The following screenshot show you can create your own version of localhost.
</div>

![Connected using a customer local host](/post/2022/connect-to-a-service-in-a-kubernetes-instance-hosted-on-wsl-from-windows/connected-to-service-using-custom-loca-host.png)

If you still cannot reach the service after having modified the host's file, flush your DNS in Windows using the following command.

```Powershell
ipconfig /flushdns
```

While this works, I did not find it to be a viable solution so I moved on to the next approach, port forwarding.
 
## Port Forwarding

This approach does not involve any modifications of host files, and there is no need to obtain the node IP. It involves using [port forwarding](https://kubernetes.io/docs/tasks/access-application-cluster/port-forward-access-application-cluster/)

After following the commands under [Connection Issue](/post/2022/connect-to-a-service-in-a-kubernetes-instance-hosted-on-wsl-from-windows#connection-issue) and having verified that all 3 pods are up and running, use the following command to port forward from the WSL's localhost to Windows' localhost. Remember, [they are now one and the same](https://learn.microsoft.com/en-us/windows/wsl/networking#accessing-linux-networking-apps-from-windows-localhost).

To start port forwarding traffic to the Nginx pods run the following command.

```bash
kubectl port-forward --address 0.0.0.0 service/nginx-deployment 31080
```

Kubernetes will forward traffic from 31080 to port 80 on the container, and since WSL's localhost:31080 is the same as Windows' 31080, I opened up the browser to localhost:31080 to connect to the service.

<div style="padding: 15px; border: 1px solid transparent; border-color: transparent; margin-bottom: 20px; border-radius: 4px; color: #8a6d3b;; background-color: #fcf8e3; border-color: #faebcc;">
The following screenshot shows that you can connect using localhost so long as you are port-forwarding the traffic in WSL.
</div>

![Connecte to the service using port-forward](/post/2022/connect-to-a-service-in-a-kubernetes-instance-hosted-on-wsl-from-windows/connected-to-the-service-using-port-forward.png)

Another approach would be to forward traffic from the pod instead of the service using the following command, the result is the same.

```Bash
kubectl port-forward nginx-deployment-7fb96c846b-4c4r4 32196:80
```

Where nginx-deployment-7fb96c846b-4c4r4 is the name of one of the three pods running.

This is great, I love being able to connect using localhost, however, this solution is temporary, as soon as you stop port-forwarding traffic, the connection will stop work working on Windows. This approach may or may not be viable for you, if you are looking for a similar solution but one that is permanent, then go directly to [hostport](/post/2022/connect-to-a-service-in-a-kubernetes-instance-hosted-on-wsl-from-windows#hostport). For now, on to the next approach, using MetalLB.

## MetalLB

This approach involves using a microk8s addon, [MetalLB](https://metallb.org/), to allow load balancing. This approach ended up being exactly as [Node IP](post/2022/connect-to-a-service-in-a-kubernetes-instance-hosted-on-wsl-from-windows#node-ip). If you don't feel like editing host files is your solution then skip this part or stick around, you will at least learn how to use MetalLB to create a load balancer. Fun!

MetalLB is a load-balancer implementation for bare metal Kubernetes clusters, using standard routing protocols. It can be enabled in microk8s using the following command.

```bash
microk8s enable metallb
```

Note that when you execute the command, metallb is going to expect you to provide an IP. You can specify it as a range like 10.64.140.43-10.64.140.49,192.168.0.105-192.168.0.111 or using CDIR notation. I prefer CDIR notation.

First, I need an IP address. If I run the following command I will get the IP of the node, which in this case is the IP of WSL.

```bash
kubectl get node -o wide
```

Gives me the following output.

```bash
NAME    STATUS   ROLES    AGE   VERSION   INTERNAL-IP     EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION                      CONTAINER-RUNTIME
gohan   Ready    <none>   19h   v1.25.4   172.23.200.34   <none>        Ubuntu 22.04.1 LTS   5.15.79.1-microsoft-standard-WSL2   containerd://1.6.9
```

For my load balancer IP, I'm going to change the last octet from 34 to 49, so the IP for metallb is going to be 172.23.200.49.

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

Just like [NodePort](post/2022/connect-to-a-service-in-a-kubernetes-instance-hosted-on-wsl-from-windows#node-ip), if don't enjoy using an IP address to access the service, modify the hosts' file and map the IP of the load balancer to a custom domain.

## HostPort

What ultimately ended up being my preferred solution. HostPort keeps everything simple, no hosts files, no IPs, and no fuss. I consider this approach to be the same as [port forwarding](/post/2022/connect-to-a-service-in-a-kubernetes-instance-hosted-on-wsl-from-windows#port-forwarding) but unlike Port Forwarding, this approach has a permanent solution, well so long as the pods are running.

HostPort applies to the Kubernetes containers. The port is exposed to the WSL network host. This is often an approach not recommended because the Host IP can change, however, I have found that it only applies to production environments. In WSL, the likelihood of the host IP changing is very small.

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
NewReplicaSet:   nginx-deployment-66cdbdf488 (3/3 replicas created)
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

After the **netsh interface** command is executed I was ready to connect. I opened a browser to localhost:5700 on the Windows machine.

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

## Localhost Loopback

As mentioned in [Node IP](/post/2022/connect-to-a-service-in-a-kubernetes-instance-hosted-on-wsl-from-windows#node-ip) I started messing around with localhost too much to the point that I realized I was going down the wrong path. I am going to document a solution that I thought would work but didn't. I am only including it in this post because it shows you what not to do. 

Again. 

<div style="width:100%;height:0;padding-bottom:52%;position:relative;"><iframe src="https://giphy.com/embed/fSkpUE72ynxPsKVNYW" width="100%" height="100%" style="position:absolute" frameBorder="0" class="giphy-embed" allowFullScreen></iframe></div>

The idea here is to add an additional IP to the localhost loopback. For example, 127.0.0.1 is the loopback address for localhost, you can add an additional address to your network interface.

To do this, open the Run app by pressing the windows key and r at the same time. When the Run app opens up, type hdwwiz.exe, and the add hardware wizard window will appear.

![Add hardware wizard](/post/2022/connect-to-a-service-in-a-kubernetes-instance-hosted-on-wsl-from-windows/add-hardware-wizrd.png)

Hit Next.

![Wizard installtion option](/post/2022/connect-to-a-service-in-a-kubernetes-instance-hosted-on-wsl-from-windows/wizard-installation-option.png)

Check the option install hardware I manually select from a list, then click next.

![Network adapters](/post/2022/connect-to-a-service-in-a-kubernetes-instance-hosted-on-wsl-from-windows/network-adapters.png)

Locate and select "Network Adapters", then click next.

![Manufacturere and model](/post/2022/connect-to-a-service-in-a-kubernetes-instance-hosted-on-wsl-from-windows/loopback-manufacturer.png)

Select "Microsoft" as the Manufacturer and select "Microsoft KM-Test Loopback Adapter", then click next, on the next window click next then finish.

Go to your network's connection and the new adapter will be listed there.

![Network connection](/post/2022/connect-to-a-service-in-a-kubernetes-instance-hosted-on-wsl-from-windows/network-connections.png)

In my case, it is listed as "Ethernet". Right-click on your newly added adapter, and click on properties.

![Network adapter properties](/post/2022/connect-to-a-service-in-a-kubernetes-instance-hosted-on-wsl-from-windows/network-adapter-properties.png)

Find "Internet Network Protocol Version v4" and click on "Properties".

On the properties windows, use the node IP as the IP Address, and for the subnet mask, use the following command.

```Powershell
ipconfig
```

![submask](/post/2022/connect-to-a-service-in-a-kubernetes-instance-hosted-on-wsl-from-windows/submask.png)

![Internet V4 Properties](/post/2022/connect-to-a-service-in-a-kubernetes-instance-hosted-on-wsl-from-windows/internet-protocol-v4-properties.png)

In the end, this didn't work. I don't know why, but if you do, I would love to hear from you.

## Conclusion

For now, I feel like HostPort is the best solution I could come up with. 

It allows me to do what I wanted, to be able to connect from Window's localhost to a service running in Kubernetes, which is hosted in WSL. If you know of a better way, please add it [here](https://github.com/circleupx/circleupx.github.io/tree/master/content/post/2022/connect-to-a-service-in-a-kubernetes-instance-hosted-on-wsl-from-windows/index.md).

Thanks for reading.