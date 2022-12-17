---
title: Remote Desktop Into WSL
tags: [WSL]
author: "Yunier"
date: "2022-11-17"
description: "A quick guide on how to RDP into a WSL instance."
series: [WSL]
---

There have been a few instances where I could not figure out a problem within WSL. Problems that would be easier to fix if I had a UI instead of just an instance of the WSL shell. If you find yourself in such a situation know that you can install the UI portion, a Linux desktop on top of WSL. Once the UI has been installed you can RDP into the WSL instance allowing you to use the WSL distro as if it were natively installed on your machine.

## Install A Linux Desktop

To get started you will need to determine which Linux Desktop environment you would like to install. You have [many options](https://geekflare.com/linux-desktop-environment/) to choose from but I recommend you stick to something that you are familiar with or something light like [xfce](https://www.xfce.org/). I will choose [Xfce](https://www.xfce.org/) as it has a small footprint.

To install the xfce desktop environment run the following commands on your WSL terminal.

```shell
sudo apt update && sudo apt upgrade -y
sudo apt install xfce4 -y
```

The first command will install available upgrades on the Ubuntu WSL instance, the second command will install xfce.

## Display Manager

During the installation, the prompt below may appear. If it does, set the display manager as gdm3 by hitting enter.

![xface prompt](/post/2022/remote-desktop-into-wsl/xfce-package-configuration.png)

### lightdm

If you pick the wrong value and you would like to switch to lightdm as the display manager then run the following command.

```shell
sudo dpkg-reconfigure lightdm
```

### gdm3

If you picked lightdm and would like to switch to gdm3 then run the following command.

```shell
sudo dpkg-reconfigure gdm3
```

## IP Address

With xfce installed I need to find out the WSP IP address. That can be done by running the following command on the WSL terminal.

```shell
hostname -I
```

![IP Address](/post/2022/remote-desktop-into-wsl/ip-address.png)

As seen in the screenshot above, the WSL IP is 172.17.113.211.

## Enable XRDP

To be able to connect to the WSL instance from an RDP client like Window's Remote Desktop Connection, you will need to enable the [xrdp service](https://en.wikipedia.org/wiki/Xrdp). It should already be installed on your machine, if it is not, then run the following command from the WSL terminal to install it.

```shell
sudo apt install xrdp -y 
```

You can verify that it is installed by checking if the service is running, execute the following command on the WSL terminal.

```shell
sudo service xrdp status
```

If installed correctly, you will get an output similar to the one shown in the screenshot below.

![xrdp](/post/2022/remote-desktop-into-wsl/rdp-output.png)

Now, start the service by running the following command from the WSL terminal.

```shell
sudo service xrdp start
```

## RDP

Open up your RDP client. Connect to the WSL IP address obtained as the output of **hostname -I**. The XRDP username and password prompt window will appear. Before you type your account credentials, close the WSL terminal, if you attempt to RDP into the WSL instance while you have the WSL terminal running may run into the following problems.

1. RDP will immediately close your connection and kick you out.
2. RDP will not close but instead will give you a black screen.

As you can see from the screenshot below, I was able to RDP into the WSL instance. I can now run WSL through the UI as if it were natively installed on my machine.

![rdp](/post/2022/remote-desktop-into-wsl/rdp.png)

## Troubleshooting

### Second User

RDP does not allow a user with an active session to RDP into the machine that has the active session. You can tell you are facing this problem if you RDP into the WSL instance, and authenticate successfully but then only see a black screen. As stated above, close any instances of the WSL shell and try again. Another option, if you feel like you need to have the WSL terminal instance running while trying to RDP, create a second user using the following command.

```shell
sudo adduser username
```

Set the username and password. RDP into the WSL instance using the new username.
