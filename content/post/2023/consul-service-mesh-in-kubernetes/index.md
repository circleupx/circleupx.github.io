---
title: Consul Service Mesh in Kubernetes
tags: [Kubernetes, Consul]
author: "Yunier"
date: "2023-05-28"
description: "Using Consul Service Mesh in Kubernetes"
series: ['Service Meshes']
---

## Introduction

I have been spending my last few week sharpening up my Kubernetes skills, one area that I focused on was how to enable and use a Service Mesh to Kubernetes. A service mesh is a layer in your infrastructure that enables control of inbound and outboard traffic. It controls the traffic of any app or service that uses the network.

[Kubernetes offers a wide range of Service Meshes](https://thechief.io/c/editorial/top-14-kubernetes-service-meshes/), in this blog post I am going to concentrate on HashiCorp's service mesh offering, Consul, though you may see other refer to it as Consul Connect, Consul Connect is a set of features that were added to Consul was in 2018 to enable service mesh support. 

These features consit of three parts, first, the Consul server component, the Consul client and the [sidecars](https://www.nginx.com/resources/glossary/sidecar/) that are deployed. The server component is responsible for persisting data, i.e. configurations, thus requiring high availability, don't run a single Consul Server in a Production environment. The client component resides in your node, it is responsible for reporting the health of each service running on your node as well as keeping track of the health of other services in other nodes. The client sends this information back to the Consul server component. Another responsibility of the client is to configure and manage all sidecars. The sidecars are responsible for intercepting inboud and outbound network traffic through a proxy, Consul leverages [Envoy](https://www.envoyproxy.io/) to achieve this feature.

I have prepared two sample applications to demo how you can configure and use Consul as a Service Mesh in Kubernetes. The first app will be a frotend application written in [Blazor](https://dotnet.microsoft.com/en-us/apps/aspnet/web-apps/blazor), the second app will be a Web API written in .NET 6 using [minimal APIs](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/minimal-apis/overview?view=aspnetcore-7.0). The Blazor application will call the Web API to get a weather forecast, there will be no data validation, authentication or authorization in the apps since the main focus of this post is to show how to use Consul.

## Web API

The Web API is written using the new Minimal APIs feature offered by .NET. The API generates a in-memory collection of [WeatherForecast](https://github.com/dotnet/aspnetcore/blob/main/src/ProjectTemplates/Web.ProjectTemplates/content/WebApi-CSharp/WeatherForecast.cs), the entire API can be configured as shown below.

```C#
namespace BackendApp
{
    public class Program
    {
        public static void Main(string[] args)
        {
            var builder = WebApplication.CreateBuilder(args);

            // Learn more about configuring Swagger/OpenAPI at https://aka.ms/aspnetcore/swashbuckle
            builder.Services.AddEndpointsApiExplorer();
            builder.Services.AddSwaggerGen();

            var app = builder.Build();
            app.UseHttpsRedirection();
            var summaries = new[]
            {
            "Freezing", "Bracing", "Chilly", "Cool", "Mild", "Warm", "Balmy", "Hot", "Sweltering", "Scorching"
            };

            app.MapGet("/weatherforecast", (HttpContext httpContext) =>
            {
                var forecast = Enumerable.Range(1, 5).Select(index =>
                    new WeatherForecast
                    {
                        Date = DateOnly.FromDateTime(DateTime.Now.AddDays(index)),
                        TemperatureC = Random.Shared.Next(-20, 55),
                        Summary = summaries[Random.Shared.Next(summaries.Length)]
                    })
                    .ToArray();
                return forecast;
            })
            .WithName("GetWeatherForecast")
            .WithOpenApi();

            app.Run();
        }
    }
}
```

In the code above, the router handler MapGet is used to map incoming HTTP request whenever the incoming router matches "/weatherforecast".

### API Docker File

To be able to run the API in Kubernetes I will need to conternarize the app, the following Docker file should do the trick.

```Shell
FROM mcr.microsoft.com/dotnet/aspnet:7.0 AS base
WORKDIR /app
EXPOSE 80

FROM mcr.microsoft.com/dotnet/sdk:7.0 AS build
WORKDIR /src
COPY ["App/BackendApp.csproj", "App/"]
RUN dotnet restore "App/BackendApp.csproj"
COPY . .
WORKDIR "/src/App"
RUN dotnet build "BackendApp.csproj" -c Release -o /app/build

FROM build AS publish
RUN dotnet publish "BackendApp.csproj" -c Release -o /app/publish /p:UseAppHost=false

FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "BackendApp.dll"]
```

Then to build the container image with Docker execute the following command.

```Shell
docker build . -f App/Dockerfile -t backend:1.0.0
```

The dot afte the build command refers to the current working directory, the -f is the path to the Docker file and the -t command sets a tag, in my case I decided to append version 1.0.0 to my app. After executing the command above, run the following command to confirm the image was created.

```Shell
$ docker image ls
REPOSITORY   TAG      IMAGE ID       CREATED          SIZE
backend      1.0.0    f20ce50a5b86   39 seconds ago   216MB
```

## Blazor App

Next up, I have created a frontend application base on Blazor, the app uses the default Blazor template, as part of the template, Blazor adds a weather forecast page. The data displayed is loaded from an in-memory collection. I am going to change the application to load the data from the Web API created above.

This is the WeatherForecastService that comes from the Blazor template.

```C#
public class WeatherForecastService
{
    private static readonly string[] Summaries = new[]
    {
    "Freezing", "Bracing", "Chilly", "Cool", "Mild", "Warm", "Balmy", "Hot", "Sweltering", "Scorching"
};
    public Task<WeatherForecast[]> GetForecastAsync(DateOnly startDate)
    {
        return Task.FromResult(Enumerable.Range(1, 5).Select(index => new WeatherForecast
        {
            Date = startDate.AddDays(index),
            TemperatureC = Random.Shared.Next(-20, 55),
            Summary = Summaries[Random.Shared.Next(Summaries.Length)]
        }).ToArray());
    }
}
```

If you were to run the app an navigate to the forecast page you would see something similar to what is being shown on the screenshot below.

![Weather Forecast Page](/post/2023/consul-service-mesh-in-kubernetes/weather-forecast-page.png)

I'll need to refactor the GetForecastAsync method to use an HTTP client to make an API request to the Web API endpoint, https://localhost:7043/weatherforecast, to retrieve the WeatherForecast data from the API

```C#
namespace FrontEndApp.Data
{
    public class WeatherForecastService
    {
        public async Task<WeatherForecast[]> GetForecastAsync()
        {
            var httpClient = new HttpClient();
            var weatherForecastCollection = await httpClient.GetFromJsonAsync<WeatherForecast[]>("https://localhost:7043/weatherforecast");
            return weatherForecastCollection;
        }
    }
}
```

The code above will establish a dependency between the Web API and the Blazor App, the two apps now communicate though the network, once I jump into Consul I will show how the network communication between these two apps can be configured.

### Blazor Docker File

Just like the Web API I will need to conternrize the Blazor app, the following Docker file should do the trick.

```Shell
FROM mcr.microsoft.com/dotnet/aspnet:7.0 AS base
WORKDIR /app
EXPOSE 80

FROM mcr.microsoft.com/dotnet/sdk:7.0 AS build
WORKDIR /src
COPY ["App/FrontEndApp.csproj", "App/"]
RUN dotnet restore "App/FrontEndApp.csproj"
COPY . .
WORKDIR "/src/App"
RUN dotnet build "FrontEndApp.csproj" -c Release -o /app/build

FROM build AS publish
RUN dotnet publish "FrontEndApp.csproj" -c Release -o /app/publish /p:UseAppHost=false

FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "FrontEndApp.dll"]
```

Then to build the container image with Docker execute the following command.

```Shell
docker build . -f App/Dockerfile -t frontend:1.0.0
```

The dot afte the build command refers to the current working directory, the -f is the path to the Docker file and the -t command sets a tag, in my case I decided to append version 1.0.0 to my app. After executing the command above, run the following command to confirm the image was created.

```Shell
$ docker image ls
REPOSITORY   TAG      IMAGE ID       CREATED          SIZE
frontend     1.0.0    f20ce50a5b86   4 seconds ago   216MB
```

## Configuring Helm 

I plan to use Helm to deploy my applications, [Helm](https://helm.sh/) is the package manager for Kubernetes, it is a great tool manage Kubernetes applications, you can use Helm to define, install and upgrade any Kubernetes application.

To use Helm I will need to create a Helm Chart for the frontend and backend application. Executing the following command will create a new helm chart.

```Shell
helm create backendapp
```

Exectuting the commmand above will create a new chart title backendapp, the chart is composed of a few files as seen on the folder tree below.

```Text
Root
├── .vscode
│   └── settings.json
├── App
│   ├── appsettings.json
│   ├── BackendApp.csproj
│   ├── Dockerfile
│   ├── Program.cs
│   ├── Properties
│   │   └── launchSettings.json
│   └── WeatherForecast.cs
├── BackendApp.sln
└── Deploy
    └── backendapp
        ├── .helmignore
        ├── Chart.yaml
        ├── charts
        ├── templates
        │   ├── deployment.yaml
        │   ├── hpa.yaml
        │   ├── ingress.yaml
        │   ├── NOTES.txt
        │   ├── service.yaml
        │   ├── serviceaccount.yaml
        │   ├── tests
        │   │   └── test-connection.yaml
        │   └── _helpers.tpl
        └── values.yaml  #The file you provide to consumers of your chart.
```

The deploy folder is the root directory for all helm charts. Within that folder, you will find the chart.yaml file, this is where all chart metadata is placed i.e. chart name, the Charts folder is where you place any Charts that your Chart depends on. The template folder is the directory containing the files used to manifest your chart. It utilizes Go's [template language](https://pkg.go.dev/text/template). 

The chart needs a few updates, first in the Chart.yaml file, I will set teh appVersion to be 1.0.0, this is because the Docker image was tagged with 1.0.0 and in my values.yaml I left the tag value empty.

```YAML
apiVersion: v2
name: backendapp
description: A Helm chart for Kubernetes
type: application
version: 0.1.0
appVersion: "1.0.0"
```

The next change required is to the values.yaml file, I need to change the repository as newly created Helm Charts look for Nginx by default. The name of my container image is backend, so that is the value I will use, see below.

```YAML
image:
  repository: backendapp
  pullPolicy: IfNotPresent
  # Overrides the image tag whose default is the chart appVekubecrsion.
  tag: ""
```

The pullPolicy being set to "IfNotPresent" is important as it allows Helm to pull image from the local container registry.

No more changes are required for the backend Helm chart, next, I will create the Helm chart for the frontend using the following command.

```Shell
helm create frontendapp
```

Just like the Web API, this will create a chart named frontendapp under my deploy folder within the FrontendApp and just like the Web API I will need to update the values.yaml and chart.yaml files as shown below.

```YAML
apiVersion: v2
name: frontendapp
description: A Helm chart for Kubernetes
type: application
version: 0.1.0
appVersion: "1.0.0"
```

```YAML
image:
  repository: frontendapp
  pullPolicy: IfNotPresent
  # Overrides the image tag whose default is the chart appVekubecrsion.
  tag: ""
```

I do need to made an additional change, all Helm charts by port forward from port 8080, the two apps could potentially run on the same node and run into a port collision, therefore, I will change the UI to port forward on 8081, you can do it manually on the terminal using the kubectl port forward but I am going to go ahead and modify the command on the notes.txt file.

## Deploying App to Kubernetes

Time to use Helm to deploy both apps. First I am going to deploy the backend app using the following command.

```Shell
helm install backendapp ./
```

The helm install command takes the name for the install, the chart, this can be a path and a set of flags, in the command above I am not providing any flags, just the name and the path to the chart, which is the current directory.

The output of that command should be the values found in notes.txt, as shown below.

```Shell
export POD_NAME=$(kubectl get pods --namespace default -l "app.kubernetes.io/name=backendapp,app.kubernetes.io/instance=backendapp" -o jsonpath="{.items[0].metadata.name}")
export CONTAINER_PORT=$(kubectl get pod --namespace default $POD_NAME -o jsonpath="{.spec.containers[0].ports[0].containerPort}")
echo "Visit http://127.0.0.1:8080 to use your application"
kubectl --namespace default port-forward $POD_NAME 8080:$CONTAINER_PORT
```

Run each of the following commands to being port forwarding from port 8080 on your machine to port 80 on the container.

```Shell
Forwarding from 127.0.0.1:8080 -> 80
Forwarding from [::1]:8080 -> 80
```

Great, I should be able to send the following curl command and get a successful response from the API. 

```Shell
curl -X 'GET' \
  'http://localhost:8080/weatherforecast' \
  -H 'accept: application/json'
```

The command above yields the following response from the API.

```Json
[
  {
    "date": "2023-06-22",
    "temperatureC": -5,
    "temperatureF": 24,
    "summary": "Bracing"
  },
  {
    "date": "2023-06-23",
    "temperatureC": 14,
    "temperatureF": 57,
    "summary": "Scorching"
  },
  {
    "date": "2023-06-24",
    "temperatureC": 15,
    "temperatureF": 58,
    "summary": "Mild"
  },
  {
    "date": "2023-06-25",
    "temperatureC": 27,
    "temperatureF": 80,
    "summary": "Chilly"
  },
  {
    "date": "2023-06-26",
    "temperatureC": 46,
    "temperatureF": 114,
    "summary": "Bracing"
  }
]
```

Excellent, the API was deployed sucessfully and it is listening to incoming request, I don't need to keep the port-forward proxy open so I'll use CTRL+C to end the proxy.

Time to deploy the frontend app.

Just like before, execute the Helm install command.

```Shell
helm install frontendapp ./
```

The command should yield something similar to the following.

```Shell
NAME: frontendapp
LAST DEPLOYED: Wed Jun 21 20:54:33 2023
NAMESPACE: default
STATUS: deployed
REVISION: 1
NOTES:
1. Get the application URL by running these commands:
   export POD_NAME=$(kubectl get pods --namespace default -l "app.kubernetes.io/name=frontendapp,app.kubernetes.io/instance=frontendapp" -o jsonpath="{.items[0].metadata.name}")
   export CONTAINER_PORT=$(kubectl get pod --namespace default $POD_NAME -o jsonpath="{.spec.containers[0].ports[0].containerPort}")
   echo "Visit http://127.0.0.1:8081 to use your application"
   kubectl --namespace default port-forward $POD_NAME 8081:$CONTAINER_PORT
```

Execute each of the command from the notes to get the port-forwading proxy going.

```Shell
Forwarding from 127.0.0.1:8081 -> 80
Forwarding from [::1]:8081 -> 80
```

Great, let's test it out, I should be able to open my browser to http://localhost:8081, then navigate to the Fetch Data page and see the randomly generate weather forecast create by the API.

![Blazor in Kubernetes](/post/2023/consul-service-mesh-in-kubernetes/blazor-in-kubernetes.png)

As expected, it works, the Blazor App is running on Kubernetes and it is able to communicate with the Web API.

Now that we have the apps deployed and exchanging date I'll introduce Consul and configure it to control how the app communicate with each other.

> **Note**: Not shown here, but the frontend required an additional change, the URL change from **https://localhost:7043/weatherforecast** to **http://backendapp:80/weatherforecast** that is because the frontend Pod talks to the backend pod though a Kubernetes Service name backendapp. This service was generated when the app was install with Helm, see the service.yaml file under the templates folder.

## Enable Consul

## Gotchas

## Conclusion