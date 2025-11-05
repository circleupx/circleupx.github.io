---
title: VSCode Configuration to attach to a container running in a kubernetes cluster
tags: [vscode, go, kubernetes]
author: "Yunier"
date: "2025-11-04"
description: "How to configure VSCode to debug containerized apps running in local kubernetes."
---

VSCode is an amazing text editor, it offers a wide variety of features and an awesome list of extensions. One feature that I like to use is "[Attach to a container in a Kubernetes cluster](https://code.visualstudio.com/docs/devcontainers/attach-container#_attach-to-a-container-in-a-kubernetes-cluster)", that is attaching my code to a running instance of my app running in my local kubernetes cluster. 

Super useful if you need to test, debug or monitor your app while it is running. I recently found myself in a situation where I had to restart the app and reattached myself to the container over and over again. It became tedious to find the pod in the Kubernetes view, right clicking on the pod once I found it to select the attach option. Having to do this over and over again prompted me to look into whether this could be easier to do via some configuration. I wanted something that would allow me to easily attach to a pod. I looked around the web and found nothing so this is what I came up with.

First, edit or create the launch.json file, under it add a new configuration as shown below. In my case I was working with a GO app that has been configured with Delve, but this configuration will work for any other that can expose a debug port like GO does through Delve

```json
{
    "version": "0.2.0",
    "configurations" : [
        {
            "name": "Attach to app", // This is the name that will appear as an option on the Run and Debug window in VSCode.
            "type": "go", // The type of service, PHP, C#, Python etc. In our case, this is a go service.
            "request": "attach", // we want to attach vs launching.
            "mode": "remote", // Tell VSCode to connect to an app running remotely, without it vscode will prompt you to pick a local process to attach to.
            "path": "/app", // where the app is running within your container. In my case my go app is running under the /app folder. 
            "port": 10000, // the port on which preLaunchTask will forward requests from the pod. I use 10000 as it is unlikely to be used by another service running on my pc.
            "host": "127.0.0.1", //localhost.
            "preLaunchTask": "Enable Port Forwarding", // name of the pre launch task, this task runs before launch.json, it is responsible for running kubectl port-forward.
            "showLog": true, // optional field, useful for debugging configuration.
            "trace": "verbose" // optional field, useful for debugging.
        }
    ]
}
```
See [Launch Settings Attributes](https://code.visualstudio.com/docs/debugtest/debugging-configuration#_launchjson-attributes) for more information. 

The second configuration required to make this work is a task.json as seen below.

```json
{
  "version": "2.0.0",
  "tasks": [
    {
      "label": "Port Forward my-service",
      "type": "shell",
      "command": "kubectl port-forward deployment/my-service 10000:2345",
      "isBackground": true,
      "problemMatcher": {
        "pattern": {
          "regexp": ".",
          "file": 1,
          "message": 0
        },
        "background": {
          "activeOnStart": true,
          "beginsPattern": "Forwarding from",
          "endsPattern": "Forwarding from"
        }
      },
      "presentation": {
        "echo": true,
        "reveal": "never",
        "focus": false,
        "panel": "dedicated"
      }
    }
  ]
}
```

This JSON snippet defines a Visual Studio Code task for port forwarding a Kubernetes deployment. The task is named "Port Forward my-service" and uses the shell type to execute the command kubectl port-forward deployment/my-service 10000:2345. This command forwards traffic from port 2345 on the my-service deployment in your Kubernetes cluster to port 10000 on your local machine, allowing you to access services running inside the cluster as if they were local.

The isBackground property is set to true, meaning the task runs in the background and does not block other tasks or editor actions. The problemMatcher configuration helps VS Code detect when the port forwarding is active by looking for output lines that match the pattern "Forwarding from". This allows the editor to track the task's status and manage its lifecycle appropriately. The phrase "Forwarding from" is part of the output from kubectl when you run the port-forward command.

The presentation section customizes how the task output is displayed. It ensures command output is echoed, the output panel is not automatically revealed, focus is not shifted to the panel, and a dedicated panel is used for this task. This setup is useful for development workflows that require remote debugging or service access through Kubernetes port forwarding, integrating seamlessly with VS Code's task system.

The result of these new configurations is this.

![VSCode](/post/2025/vscode-configuration-to-attach-to-a-container-running-in-a-kubernetes-cluster/VSCode.png)

Now I can simply press F5 and quickly attach my code to the running app.