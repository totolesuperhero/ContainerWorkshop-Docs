# Lab 5 - Working with environments

In this lab you will learn how to create and run compositions.

Goals for this lab:

- [Working with compositions and Docker Compose](#work)
- [Create compositions for different environments](#create)
- [Change implementation to work with environment variables](#change)

> Make sure to configure Docker Desktop to run Linux containers.
>
> This lab uses files from the `Code` repository.

## Prerequisites

Make sure you have completed [Lab 1 - Getting Started](Lab1-GettingStarted.md).

## <a name="work"></a>Working with compositions and Docker Compose

Compositions are essential for managing the many different combinations of containers, images, run-time details and environmental settings. Typically an application consists of multiple running containers. Managing each of these individually is both difficult and labor intensive. Moreover, it does not capture the relationships and dependencies that exist.

Docker Compose is the tool of choice for this lab to manage compositions of containers. It allows you to use a command-line interface, similar to the Docker CLI, to interact with compositions defined in a `docker-compose.yml` file. There are other tools that allow the creation of compositions, such as the YAML files of Kubernetes. You will use Docker Compose in this lab.

To become familiar with Docker Compose, you will first start a container based on a YAML file using `docker-compose.exe`. Create a file named `docker-compose.ci.build.yml` in the root of your solution and add the following content to it:

```yaml
version: "3.4"

services:
  ci-build:
    image: mcr.microsoft.com/dotnet/sdk:6.0
    volumes:
      - .:/src
    working_dir: /src
    command: /bin/bash -c "dotnet restore ./ContainerWorkshop.sln && dotnet publish ./ContainerWorkshop.sln -c Release -o ./obj/Docker/publish"
```

The definitions in the compose file describe a service called `ci-build` that uses the image `mcr.microsoft.com/dotnet/sdk:6.0` and has a volume mapping to the root of the source code. The command starts a build in the working directory `src`.

Start this composition by executing the command from the root of the Visual Studio solution where the Docker Compose YAML files are located:

```cmd
docker-compose -f docker-compose.ci.build.yml up
```

The command will 'up' (meaning 'start') the composition and perform a build and publish of the projects in the solution `ContainerWorkshop`.

You could use this composition in your build pipeline to perform the build and publishing of the binaries required to create the container images of the solution.

There is also a new way to accomplish the same thing. This is by using multi-stage builds in the Dockerfile. Instead of running a new composition that spins up a container to build a container, you can create a Dockerfile that uses stages to build your application. As an example look at the following file:

```docker
FROM mcr.microsoft.com/dotnet/aspnet:6.0 AS base
WORKDIR /app
EXPOSE 80
EXPOSE 443

FROM mcr.microsoft.com/dotnet/sdk:6.0 AS build
WORKDIR /src
COPY ["src/GamingWebApp/GamingWebApp.csproj", "src/GamingWebApp/"]
RUN dotnet restore "src/GamingWebApp/GamingWebApp.csproj"
COPY . .
WORKDIR "/src/src/GamingWebApp"
RUN dotnet build "GamingWebApp.csproj" -c Release -o /app/build

FROM build AS publish
RUN dotnet publish "GamingWebApp.csproj" -c Release -o /app/publish

FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "GamingWebApp.dll"]
```

This file creates a clean base image, that is based on the runtime only. Next, it uses a image with the SDK to build the application. Then it publishes the build and in the final stage it uses the `COPY --from` syntax to only use the output of the previous stage in your clean base image. [Read more about multi-stage builds here.](https://docs.docker.com/develop/develop-images/multistage-build/)

Multi-stage builds are used by the Docker tooling in Visual Studio 2022 by default, so you should be able to find similar content inside the `Dockerfile` files of both ASP.NET Core projects.

## <a name="create"></a>Create compositions for different environments

One of the useful features of Docker Compose is the layering and cascading of multiple YAML compose files. With it you can introduce concepts such as base compositions, inheritance and overrides.

By default the Docker Compose tooling assumes that your main composition file is called `docker-compose.yml`. Executing `docker-compose <command>` will look for that particular file. Try to run a build that way.

```cmd
docker-compose build
```

It is convenient when your `docker-compose.yml` file is able to build the container images of the composition. The build of the container images is defined in the `Dockerfile` in the root of the individual projects and uses multiple stages to create a clean final image.

Make sure you understand the `docker-compose.yml` contents.

The Docker support in Visual Studio 2022 makes a similar assumption. It assumes that runtime details for compositions started from Visual Studio are defined in `docker-compose.override.yml`. Open that file which is located underneath the `docker-compose.yml` file in the Solution Explorer tree of Visual Studio.

The combination of the two aforementioned compose files is enough to start a composition. You will need to specify both files in the command in the correct order.

```cmd
docker-compose -f docker-compose.yml -f docker-compose.override.yml up
```

Ideally, your override file for Visual Studio contains the services and settings that are needed when running from the IDE on a development machine.

Take a moment to contemplate whether the `sql.data` service should be defined in the `docker-compose.yml` file or elsewhere. Remember that running SQL Server from a container is not recommended in a production scenario unless special measures have been taken. For local development purposes it might be useful to have a SQL Server instance that loses its data on each start of the hosting container. You can also choose to use a volume mapping to avoid data loss.

Change the location of the definition to the override compose file. Merge it with the existing service.

Enhance the override file by adding the dependencies of the web application on the web api, and the web api on the sql.data service. For each of the two dependent services `gamingwebapp` and `leaderboardwebapi` add a `depends_on` naming the dependency by service name. For example:

```yaml
depends_on:
  - "leaderboardwebapi"
```

In essence, the `docker-compose.override.yml` is the development environment override file by convention.

## <a name="change"></a>Working with environments in .NET

> Switch back to the `master` branch to make sure all files so far are up to date. Make sure to commit or undo any changes you made to your `start` branch before switching.

In this sample application, the web application only has a single setting for an external Web API endpoint.

```yaml
- LeaderboardApiOptions__BaseUrl=http://leaderboardwebapi
```

This way you can formalize a group of related settings, regardless of their origin. It can be from one of the `appsettings.json` files, `docker-compose.override.yml` files or even environment variables.

In the web application project, you will find a class called `LeaderboardApiOptions` with a single `string` property called `BaseUrl`. In more complex scenarios, this class would likely contain more settings. For example, configuration settings for authorization and authentication.

Next, go to the `Startup` class and locate the statement in the `ConfigureServices` method that loads the web app settings from the configuration.

```c#
services.Configure<LeaderboardApiOptions>(Configuration.GetSection(nameof(LeaderboardApiOptions)));
```

This instructs the ASP.NET Core dependency injection system to add an instance of the `LeaderboardApiOptions` class to the list of registered mappings. It allows you to inject the settings into any other object created by the DI system.

Open the `Index.cshtml.cs` file and locate the constructor of the controller class. It has two parameters, which will be injected:

```c#
public IndexModel(IOptionsSnapshot<LeaderboardApiOptions> options, ILoggerFactory loggerFactory)
```

In addition, a read-only field holds the value of the injected `options` parameter values.

The last step is to use the values from the settings object at the appropriate place. Find the `Get` async method and see how the value of the settings is used in the creation of the proxy object.

```c#
public async Task OnGetAsync()
{
  Scores = new List<HighScore>();
  try
  {
    ILeaderboardClient proxy = RestService.For<ILeaderboardClient>(options.Value.BaseUrl);
  ...
}
```

While you are looking at this, follow the code to the proxy implementation and give it another examination.

## Wrapup

In this lab, you have examined the way environments can be used to distinguish various hosting situations for your Docker composition. It is important to know which settings must be changeable for different environments, as the Docker images that you build cannot be changed internally.

Continue with [Lab 6 - Kubernetes](Lab6-Kubernetes.md).
