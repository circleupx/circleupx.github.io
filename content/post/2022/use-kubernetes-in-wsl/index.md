---
title: Use Kubernetes In WSL
tags: [Kubernetes, Microk8s, WSL]
author: "Yunier"
date: "2022-11-03"
description: "Configure WSL for local development with Kubernetes 2022 edition."
series: [WSL]
---

If you find yourself in need of having to use Kubernetes in WSL, know that it is possible, hard, but possible. It might require upgrading your machine to Windows 11 if you are on Windows 10 and a few other packages.

## Prerequisite

To get started we need to know what version of Windows you are on. This is important because Kubernetes will be installed using Microk8s, which requires having snap installed and working. Snap won't work in older Windows builds.

As of November 2022, I recommend being on Windows build version 22621 or higher. You can find out what version you are on by running the following command on Powershell

```Powershell
systeminfo | findstr /B /C:"OS Name" /C:"OS Version"
```

It should output something similar to the following snippet.

```text
OS Name:    Microsoft Windows 11 Home
OS Version: 10.0.22621 N/A Build 22621
```

Next, install or have WSL installed, you can follow the official guide [Install Linux on Windows with WSL](https://learn.microsoft.com/en-us/windows/wsl/install). Do not install a distribution through WSL, instead install your desired distribution using Windows Store. This is important as the distro on the Windows Store tends to work better.

With WSL installed, confirm you are on the right version using the following command.

```Powershell
wsl -v
```

It should output something similar to the following snippet.

```text
WSL version: 0.70.4.0
Kernel version: 5.15.68.1
WSLg version: 1.0.45
MSRDC version: 1.2.3575
Direct3D version: 1.606.4
DXCore version: 10.0.25131.1002-220531-1700.rs-onecore-base2-hyp
Windows version: 10.0.22621.674
```

As of November 2022, you should be on version 0.70.4.0 or higher. If you run the command above and don't see an output or nothing happens, then it means you are on an old WSL version, you will need to update your version of WSL. You can do so by running the following command from Powershell.

```Powershell
wsl --update
```

Time to install a distribution, simply open the Windows Store and find the distro you would like to use. Personally, I recommend sticking to an Ubuntu-based distro. For this post, I will use Ubuntu 20.04.5 LTS.

Once you have the distro installed verify you have an internet connection from within WSL by running the following ping command.

```shell
ping google.com
```

### Internet

If the command above failed then it means you can't reach the internet from WSL. Please read my [Connect To The Internet From WSL](/post/2022/connect-to-the-internet-from-wsl) post before continuing.

### Snap

Time to verify which version of Snap is installed on WSL. Run the following commands.

First, update the WSL instance with the latest packages which include the latest version of snap that as of November 2022 is version  2.57.5+20.04.

```shell
sudo apt update && sudo apt upgrade -y
```

Then confirm you have snap version  2.57.5+20.04 or higher by running the following command.

```shell
snap --version
```

You should get something similar to the following code snippet.

```shell
snap                         2.57.5+20.04
snapd                        unavailable
series                       16
Windows Subsystem for Linux  -
kernel                       5.15.68.1-microsoft-standard-WSL2 (amd64)
```

With Snap, all the great apps in the Linux ecosystem are in our hands. You can browse available snaps on [snapcraft](https://snapcraft.io/).

## Systemd

The last WSL configuration needed is systemd, it can be enabled on WSL using the following command. In the future, this setting should be automatically enabled for you when you install WSL.

First, use nano to open wsl.conf

```shell
sudo nano /etc/wsl.conf
```

With wsl.conf opened, add the following lines.

```shell
[boot]
systemd=true
```

Close nano, exist the WSL instance, and execute the following code in a Powershell terminal to restart WSL.

```Powershell
wsl --shutdown
```

## Microk8s

Now that the WSL instance has been properly configured, Microk8s can be installed. Microk8s is a lightweight Kubernetes distribution that is designed to run on local systems. It was developed by Canonical, the same company that develops Ubuntu.

To install run the following command.

```shell
sudo snap install microk8s --classic
```

With microk8s installed, we can configure the microk8s alias for kubectl using the following commands.

```shell
sudo snap alias microk8s.kubectl kubectl
sudo usermod -a -G microk8s "WSL Username"
newgrp microk8s
```

Where "WSL Username" is the username of your WSL instance, it will be in your WSl terminal before the @ symbol. If you are not sure run the following command.

```shell
whoami
```

In my case, the output is the word "bleach", so the command to update the alias is as follows.

```shell
sudo usermod -a -G microk8s bleach
```

After running all of the commands above you should be able to run kubectl without microk8s, confirm by running the following command.

```shell
kubectl get node
```

The command should output something similar to the following code snippet.

```shell
NAME    STATUS   ROLES    AGE   VERSION
gohan   Ready    <none>   15m   v1.25.3
```

If so, then congratulations, you have successfully installed and configured Kubernetes on WSL.

A quick note, WSL shares the same localhost as your Windows machine, this is why you can run an app on localhost on WSL and be able to access it on your windows machine. With Kubernetes, there is an additional configuration required to get this working. I will cover those configurations in a [future post](/post/2022/connect-from-windows-to-kubernetes-on-wsl/).
