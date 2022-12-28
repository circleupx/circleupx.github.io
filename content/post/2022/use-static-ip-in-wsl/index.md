---
title: Use Static IP In WSL
tags: [Kubernetes, Microk8s, WSL]
author: "Yunier"
date: "2022-12-23"
description: "Configure WSL to use a static IP address."
series: [WSL]
---

In my last post, [Kubernetes In WSL - Connect to a service from Windows](/post/2022/connect-to-a-service-in-a-kubernetes-instance-hosted-on-wsl-from-windows/), I explored a few ways to connect to a Kubernetes service from the host machine, Windows. In the end of that blog post, I stated that using [HostPort](post/2022/connect-to-a-service-in-a-kubernetes-instance-hosted-on-wsl-from-windows/#using-hostport) was the best option because at the time I did not know how to assign a static IP address to WSL. 

Without using a static IP address, when WSL is restarted a new IP is assigned. Having a dynamic IP made it harder for me to connect to a Kubernetes service from Windows as I would need to update all my configurations whenever a new IP was assigned to WSL.

Luckily I was not the only developer that wanted WSL to use a static IP. In [WSL2 Set static ip?](https://github.com/microsoft/WSL/issues/4210), a few developers outlined how you can create a [Hyper-V switch](https://learn.microsoft.com/en-us/windows-server/virtualization/hyper-v-virtual-switch/hyper-v-virtual-switch) to bridge the network between WSL and Windows. By using a Hyper-V switch, the WSL IP becomes static, and by having a static IP, I can create a load balancer that uses that static IP to serve traffic.

In today's post, I want to demonstrate how to create a [Hyper-V switch](https://learn.microsoft.com/en-us/windows-server/virtualization/hyper-v-virtual-switch/hyper-v-virtual-switch), how to assign it to WSL distro and how to add a load balancer to a Kubernetes service that uses the static IP.

## Prerequisite

Before I get started I wanted to point out some requirements. First, you need to be on Windows 11 Enterprise or Pro, Hyper-V is not available in Windows 11 Home, though you can try to enable it using the method documented in [this](https://www.makeuseof.com/install-hyper-v-windows-11-home/) post. As far as I know, what I am about to show you does not work on Windows 10 or older Windows systems.

Below you will find my current system spec, you should be using the same version or higher.

```Powershell
OS Name:         Microsoft Windows 11 Pro
OS Version:      10.0.22621 N/A Build 22621
```

and my WSL version, you should be using the same version or higher.

```Powershell
WSL version: 1.0.3.0
Kernel version: 5.15.79.1
WSLg version: 1.0.47
MSRDC version: 1.2.3575
Direct3D version: 1.606.4
DXCore version: 10.0.25131.1002-220531-1700.rs-onecore-base2-hyp
Windows version: 10.0.22621.963
```

## Hyper-V Switch

The first step in using a Hyper-V switch is to enable the feature in Windows. Open Powershell as an admin and run the following command.

```Powershell
DISM /Online /Enable-Feature /All /FeatureName:Microsoft-Hyper-V /all
```

After the command has been successfully executed you will be prompted to restart Windows, do so.

Once Windows fully boots up, search the Hyper-V manager on the start menu and open it.

![Hyper-V Manager](/post/2022/use-static-ip-in-wsl/hyper-v-windows-menu.png)

Inside the Hyper-V manager, under Actions, there should be an option called "Virtual Switch Manager", click on that option

![Virtual Switch Manager](/post/2022/use-static-ip-in-wsl/virtual-switch-manager.png)

The Virtual Switch Manager window will now appear. Under "Create virtual switch" make sure to have the "External" option selected, then click the "Create Virtual Switch" button.

![Create Virtual Switch](/post/2022/use-static-ip-in-wsl/create-virtual-switch-manager-window.png)

On the next screen, you will need to provide a name for the switch, the name is important as we will use it later on. The External Network has been chosen, I would leave it alone, just make sure that the checkbox "Allow management operating system to share this network adapter" is checked, then click on the "Apply" button.

![Switch Properties](/post/2022/use-static-ip-in-wsl/switch-properties.png)

A new window will appear giving a warning about the network being disrupted due to this change. Click "Yes" to continue.

After the Switch has been created, click the OK button.

Congratulations, you have successfully configured your Hyper-V switch. You can confirm by going to the networks view, there should be a new adapter listed called Network Bridge as seen in the image below.

![Network View](/post/2022/use-static-ip-in-wsl/network-view.png)

## WSL

Now that the network bridge has been created WSL needs to be updated to use the bridge. On windows use File Explorer to navigate to %UserProfile%, this is where the file .wslconfig needs to be placed. In case you are not aware, the file .wslconfig is used to configure global settings for WSL regardless of distro.

If you already have a .wslconfig file, open it up, if not, create a new file. Add the following content at the end of the file.

```Bash
[wsl2] 
networkingMode=bridged 
vmSwitch=WSL
```

Where WSL, the name of the switch created in the Hyper-V manager is assigned to vmSwitch. If you don't remember the name open Hyper-V manager again or run the following command from Powershell as an admin.

```Shell
Get-VMSwitch -SwitchType External | Select Name, SwitchType, NetAdapterInterfaceDescription, AllowManagementOS
```

Here is the result of that command on my system.

```Bash
Name SwitchType NetAdapterInterfaceDescription                                     AllowManagementOS
---- ---------- ------------------------------                                     -----------------
WSL    External Killer(R) Wi-Fi 6 AX1650x 160MHz Wireless Network Adapter (200NGW)              True
```

After applying the new settings to the .wslconfig file restart WSL using the following command from Powershell

```Powershell
wsl --shutdown.
```

Open a new WSL shell instance and run the following command.

```Bash
ip a 
```

In the output, look for the value of eth0.

![IP A Output](/post/2022/use-static-ip-in-wsl/ip-a-output.png)

This is the new static IP that WSL will have. You can confirm that the IP does not change by restarting WSL again or by shutting down Windows. 

Great, with a static IP we can now configure the Kubernetes load balancer.

## Kubernetes

<div style="padding: 15px; border: 1px solid transparent; border-color: transparent; margin-bottom: 20px; border-radius: 4px; color: #8a6d3b;; background-color: #fcf8e3; border-color: #faebcc;">
Follow my <a href="https://www.yunier.dev/post/2022/use-kubernetes-in-wsl/">Use Kubernetes In WSL</a> to install Kubernetes in WSL.
</div>

As shown on my [Kubernetes In WSL - Connect to a service from Windows](/post/2022/connect-to-a-service-in-a-kubernetes-instance-hosted-on-wsl-from-windows/) we can use [MetalLB](/post/2022/connect-to-a-service-in-a-kubernetes-instance-hosted-on-wsl-from-windows/#using-metallb) to create a load balancer.

First, grab the WSL IP, for me, it is now 192.168.2.10, I will increment the last octet by 1 to not conflict with the node's IP address.

In WSL, enable MetalLB using the following command.

```Bash
microk8s enable metallb:192.168.2.11/32
```

/32 at the end of the IP is used here to keep the load balancer IP static. If you don't understand why then may I suggest watching [Understanding CIDR Ranges and dividing networks](https://www.youtube.com/watch?v=MmA0-978fSk)

![MetalLb is ready](/post/2022/use-static-ip-in-wsl/metallb-is-ready.png)

Once MetalLB is ready, you can serve traffic to a load balancer. 

For example, take the following YAML obtained from the official Kubernetes docs.

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

You can deploy the Nginx deployment by running the following command.

```bash 
kubectl apply -f https://k8s.io/examples/controllers/nginx-deployment.yaml
```

Then expose the service created using the following command.

```bash
kubectl expose deployment nginx-deployment --port=31080 --target-port=80 --type=LoadBalancer
```

If you run the following command.

```Bash
kubectl get svc -o wide
```

You will see that the Nginx service created was assigned an external IP, the load balancer's IP.

![Service IP](/post/2022/use-static-ip-in-wsl/svc-ip.png)

Moment of truth, can we reach this Nginx service running on http://192.168.2.11:31080 from Windows.

![Nginx Service Windows Access](/post/2022/use-static-ip-in-wsl/nginx-service-windows-access.png)

**We can!** The service can be reached from Windows.

As mentioned in [Kubernetes In WSL - Connect to a service from Windows](post/2022/connect-to-a-service-in-a-kubernetes-instance-hosted-on-wsl-from-windows/#using-metallb) we can change the Windows host file by adding the load balancer IP and mapping to a domain. Remember you cannot map the load balancer IP to localhost but you could map it to something like www.localhost.com or local.host. If you really wish to access the Kubernetes service using localhost then may I suggest using a port proxy from Windows to WSL.

## localhost

To use localhost in Windows, you can create a port proxy from Windows to WSL using the following command from an admin shell.

```Powershell
netsh interface portproxy add v4tov4 listenport=31080 listenaddress=0.0.0.0 connectport=31080 connectaddress=192.168.2.11
```

Here, I'm port proxying traffic on port 31080 on Windows localhost, 0.0.0.0 to port 31080 on 192.168.2.11, my load balancer IP. This will allow me to access the service using http://localhost:31080/ from Windows as shown on the screen below.

![Port Proxy](/post/2022/use-static-ip-in-wsl/port-proxy-to-localhost.png)

## Conclusion

Having a static IP on WSL makes Kubernetes development in WSL so much easier, being able to connect a service running on Windows to a service running on Kubernetes is an awesome feature to have.

Quick warning, if you restart Windows, the configuration above will continue to work but I have noted that Kubernetes takes a few minutes to accept traffic again from Windows. Not exactly sure as to why, if you are in hurry simply open WSL and ping the service directly from within WSL.

Hope this post helped you get load balancing working for Kubernetes in WSL.

This will be my last post for 2022, see you next year.