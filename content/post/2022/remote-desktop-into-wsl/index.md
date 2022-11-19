---
title: Remote Desktop Into WSL
tags: [WSL]
author: "Yunier"
date: "2022-11-17"
description: "A quick guide on how to RDP into a WSL instance."
series: [WSL]
---

There have been a few instances where I could not figure out a problem in WSL. Problems would be easier to fix if I had a UI instead of an instance of the WSL shell to see what errors were occurring inside WSL or to see why an app wasn't running. If you find yourself in such a situation know that you can install the UI portion, the desktop on top of WSL. Once the UI has been installed you can RDP into the WSL instance. This will allow you to use the WSL distro as if it were natively installed on your machine.

## Install A Linux Desktop

To get started you will need to determine which Linux Desktop environment you would like to install. You have [many options](https://geekflare.com/linux-desktop-environment/) to choose from but I recommend you stick to something that you are familiar with or something light like [xfce](https://www.xfce.org/). I will choose [Xfce](https://www.xfce.org/) as I found it to be the less buggy in WSL & has a smaller footprint.

```shell
sudo apt update && sudo apt upgrade -y
sudo apt install xfce4-y
```

These commands will update Ubuntu, my WSL instance and install xfce.

### GDM3

During the installation, the prompt below may appear. If it does, set the display manager as gdm3 by hitting enter.

![xface prompt](/post/2022/remote-desktop-into-wsl/xfce-package-configuration.png)

If you pick the wrong value or need to back to this screen run the following command.

```shell
sudo dpkg-reconfigure gdm3
```

### Second User

I have noticed that on some machines I can RDP with the same username and password, while on other machines I cannot. The trick, in that case, is to create a second user for RDP. That kinda makes sense, why would an already authenticated user be able to RDP?

Anyways, add a new user by running the following command.

```shell
sudo adduser username
```

Set the username and password.

## IP Address

With xfce installed I need to find out the WSP IP address. That can be done by running the following command on the WSL terminal.

```shell
ip a
```

![IP Address](/post/2022/remote-desktop-into-wsl/ip-address.png)

As seen in the screenshot above, the WSL IP is 172.28.0.24.

## Enable XRDP

To be able to connect to the WSL instance from an RDP client like Window's Remote Desktop Connection, you will need to enable the [xrdp service](https://en.wikipedia.org/wiki/Xrdp). It should already be installed on your machine, if it is not, then run the following command from the WSL terminal to install it.

```shell
sudo apt install xrdp -y 
```

You can verify that it is installed by running the following command on the WSL terminal.

```shell
sudo systemctl status xrdp
```

If installed correctly, you will get an output similar to the one shown in the screenshot below.

![xrdp](/post/2022/remote-desktop-into-wsl/rdp-output.png)

Now run the service by running the following command from the WSL instance.

```shell
sudo service xrdp start
```

## RDP

Open up your favirote RDP client. Connect to the WSL IP address, you should see the XRDP username and password window. Attempt to login in using your credentials, if you get a black screen and an immediate exit then you will probably need to create an RDP account as mentioned above to continue. In my case, on my personal laptop, I need to use the RDP account while having the WSL instance running while on another machine I can log in using the main account. Haven't figured that out yet, if I find a solution I will post it here.

With everything configured, I can RDP into the WSL instance, as you can see from the screenshot below I was greeted by the xface desktop
![rdp](/post/2022/remote-desktop-into-wsl/rdp.png)
