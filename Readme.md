# Automating Docker Enterprise with Azure DevOps

## Introduction

Microsoft Azure DevOps (ADO) is a fully managed suite of tooling that empowers developers and operators to implement DevOps practices. The Docker Enterprise Platform can integrate with such services through the use of Azure Pipelines for automated building and deploying of container-based workloads. 

This guide does not attempt to recreate the excellent Azure DevOps documentation available online, but will focus on integration points between the two systems.

## Setting up Azure DevOps

An Azure DevOps tenant is necessary to use the service. These accounts are available from a myriad of different sources from Microsoft Developer Network (MSDN) subscriptions, to simply signing up for a free account at `$URL`

Once a tenant is secured, create a Project to hold code and pipeline definitions.

### Creating a Project

### Configuring Source Control Management

Azure DevOps is capable of working with git repositories hosted in ADO itself, or in a variety of other locations such as GitHub, or on-premises servers. Create a new git repository in the ADO Project, or link to an existing repository to begin the enablement of build automation.

Automated triggers can be used between the git repository and ADO, meaning that when a `git commit push` occurs then an ADO Pipeline can be automatically initiate. This effectively established a continuous integration (CI) capability for source code.

Once the git repository is linked with ADO, ensure that the application has been commited.

## Building a Pipeline

Pipelines in Azure DevOps define a series of steps the sequentially build, test, and package applications in various forms. 

Pipelines can be generated via two techniques. The Classic experience is a GUI-driven wizard where boxes and dropdowns are completed with pipeline steps. This system has been largely replaced with a YAML-based system more inline with other offerings in the DevOps market. 

The YAML-based pipelines offer "pipelines as code" benefits, as they are commited to source control and able to versioned and shared like any other file. This guide will focus on YAML-based pipelines.

### Triggers

Pipelines are initiated via triggers, which contain the logic that determines how and when a pipeline beings execution. Triggers can be configured in a variety of ways:

* **Manual** will require that an operator initiate a new build

* **Automatic** will initiate a build any time that a commit is added to the designated source control system. To avoid builds when non-application changes are made to the repository, specifcy a path and/or branch to specifically target application code changes.

  ```yaml
  trigger:
    branches:
      include:
      - master
    paths:
      include:
      - moby/src/*
  ```

### Pools

When a build is triggered it is added to a queue for a given agent pool. The next available agent will then have the build assigned, and it will execute the pipeline steps.

By default ADO uses a hosted agent pool where all servers are maintained by Microsoft. Alternatively, a pool of custom agents may also be used. Please see the Agents section for more detailed information on build agent setup.

Using the Ubuntu-based hosted agent (which includes a Moby-based container runtime):

```yaml
pool:
  vmImage: 'ubuntu-latest'
```

Using a pool of custom build agents:

```yaml
pool:
  name: 'UCP Agents - Linux'
```

### Steps

The steps within a pipeline define which actions are done to the source code. These actions may be defined as custom scripts, or configured via pre-built tasks.

Script blocks are used to run shell code as a pipeline step. For small scripts it is fine to place these inline within the YAML. For larger scripts consider creating a `scripts` directory within the code repository and creating dedicated `.sh`, `.ps1`, etc. files. These files may then be called from the pipeline step without cluttering the pipeline file.

Build a Docker Image with a shell script:

```yaml
scripts:
- script: |
    docker build \
      --tag $(DOCKER_REGISTRY_IMAGE):"$(git rev-parse --short HEAD)" \
      ./moby/pipeline
  displayName: 'Build Docker Image'
```

Build a Docker Image with a PowerShell scipt:
```yaml
scripts:
- script: |
    docker build `
      --tag $(DOCKER_REGISTRY_IMAGE):"$(git rev-parse --short HEAD)" `
      .\moby\pipeline
  displayName: 'Build Docker Image'
```

Tasks are pre-built actions that can easily be integrated into a pipeline. A series of tasks are available out of the box from Microsoft, however the system is also extensible through the Visual Studio Marketplace community. 

Build a Docker Image with the Docker Task:

```yaml
scripts:
- task: TODO
```

### Variables

Dockerfiles are often static assets, requiring a developer to commit code changes to adjust its behavior. Hard-coded values also impede the ability to reuse a Dockerfile across multiple contexts or image variations. The Azure DevOps platform offers variables to be defined for a given Pipeline, radically increasing the flexbility of Dockerfiles by not requiring code changes to reuse a given file. 

While variables may be named any value, it is recommended to decide upon a naming convention that promotes consistency and predictability. `DOCKER_` is one such convention prefix that clearly denotes that a variable is related to Docker. For example, `DOCKER_REGISTRY_USERNAME` would denote first that a value is related to Docker, that is is used to interact with a Registry, and that it contains an account username.

Values that may contain sensitive or secret information can make use of a Build Secret, rather than a Build Variable. When creating a variable, simply select the lock icon to convert the value to a secret. Secrets are not echo'd out in logs or allowed to be seen once set. Setting the password or token used to authenticate with a Docker Registry via a `DOCKER_REGISTRY_TOKEN` secret would be advisable instead of a variable.

## Preparing a Dockerfile

A developer working with a Dockerfile in their local environment has different requirements than a build automation system using the same file. A series of adjustments can optimize a Dockerfile for build performance, and enhance the flexibility of a file to be utilized in multiple build variations.

### Build Arguments

The mechanism to dynamically pass a value into a Dockerfile at `docker build` time is the `--build-arg` flag. A variable or secret can be used with the flag to change a build outcome without commiting a code change into the source control system. To utilize the flag we add an `ARG` line to our DOckerfile for each variable to be passed. 

For example, to dynamically expose a port with in the Dockerfile we would adjust:

```Dockerfile
FROM mcr.microsoft.com/dotnet/core/sdk:2.1
EXPOSE 80
WORKDIR /app
```

to include an `ARG`

```Dockerfile
FROM mcr.microsoft.com/dotnet/core/sdk:2.1
ARG DOCKER_IMAGE_PORT=80
EXPOSE ${DOCKER_IMAGE_PORT}
WORKDIR /app
```

Note that we set a default value by including `=80`; this value will be used if a dynamic value is not passed in at build time.

A base image may also be made into a dynamic value, however the `ARG` must be placed outside of the `FROM` statement. For example to adjust:

```Dockerfile
FROM mcr.microsoft.com/dotnet/core/sdk:2.1
EXPOSE 80
WORKDIR /app
```

would place the `ARG` at the beginning of the file:

```Dockerfile
ARG BASE_IMAGE_BUILD='mcr.microsoft.com/dotnet/core/sdk:2.1'

FROM ${BASE_IMAGE_BUILD}
EXPOSE 80
WORKDIR /app
```

Positioning the `ARG` outside of the `FROM` statement(s) places it in a higher scope than within any specific stage. 

Using the `ARG` and `--build-arg` pattern is useful to easily patch a given image when improvements are made to its base image. Adjusting a build variable and initiating a build brings the newer base image tag in without requiring a code commit.

### Labels

Metadata may be added to a Dockerfile via the `LABEL` keyword. Labels can help designate particular application owners, points of contact, or extraneous characteristics that would benefit from embedding within an image. 

Some common settings in Dockerfiles include:

```Dockerfile
FROM mcr.microsoft.com/dotnet/core/sdk:2.1
LABEL IMAGE_OWNER='Moby Whale <mobyw@docker.com>'
LABEL IMAGE_DEPARTMENT='Digital Marketing'
LABEL IMAGE_BUILT_BY='Azure DevOps'
EXPOSE 80
WORKDIR /app
```

Combine `LABEL` with `ARG` to dynamically pass metadata values into an image at build time.

```Dockerfile
FROM mcr.microsoft.com/dotnet/core/sdk:2.1
ARG COMMIT_ID=''
LABEL GIT_COMMIT_ID=${COMMIT_ID}
```

TODO: convert to powershell
```bash
docker build \
  --build-arg COMMIT_ID="$(git rev-parse --short HEAD)" \
  --tag $(DOCKER_REGISTRY_IMAGE):"$(BUILD_ID)" \
  .
```

Values for `LABEL` can be viewed in a container registry such as Docker Trusted Registry (DTR), or from the Docker CLI:

```bash
$ docker inspect moby:1
[
    {
        ...
        "Config": {
            ...
            "Labels": {
                "GIT_COMMIT_ID": "421b895"
            }
        }
        ...
    }
]
```

### Multi-Stage Builds

Dockerfiles originally functioned as a single "stage", where all steps took place in the same context. All libraries and frameworks necessary for the Dockerfile to build had to be loaded in, bloating the size of the resulting images. Much of this image size was used during the build phase, but was not necessary for the application to properly run; for example after an application compiled it does not have to have the compiler or SDK within the image to run. 

The introduction of "multi-stage builds" introduced splitting out of individual "stages" within one physical Dockerfile. In a build system such as Azure DevOps, we can define a "builder" stage with a base layer containing all necessary compilation components, and then a second, lightweight "runtime" stage devoid of hefty SDKs and compilation tooling. This last stage becomes the built image, with the builder stage serving only as a temporary intermediary. 

```Dockerfile
#=======================================================
# Stage 1: Use the larger SDK image to compile .NET code
#=======================================================
FROM mcr.microsoft.com/dotnet/core/sdk:2.1 AS build
WORKDIR /app

# copy csproj and restore as distinct layers
COPY *.sln .
COPY aspnetapp/*.csproj ./aspnetapp/
RUN dotnet restore

# copy everything else and build app
COPY aspnetapp/. ./aspnetapp/
WORKDIR /app/aspnetapp
RUN dotnet publish -c Release -o out

#=========================================================
# Stage 2: Copy built artifact into the slim runtime image 
#=========================================================
FROM mcr.microsoft.com/dotnet/core/aspnet:2.1 AS runtime
EXPOSE 80
WORKDIR /app
COPY --from=build /app/aspnetapp/out ./
ENTRYPOINT ["dotnet", "aspnetapp.dll"]
```

Using multi-stage builds in your pipelines has considerable impact to the speed and efficiency of builds, and the resulting file size of the built container image. For a .NET Core application the `aspnet` base layer without the `sdk` components is substantially smaller:  

```bash
$ docker image ls \
  --format '{{.Repository}}:{{.Tag}} -- \  {{.Size}}' | grep mcr
mcr.microsoft.com/dotnet/core/sdk:2.1 -- 1.74GB
mcr.microsoft.com/dotnet/core/aspnet:2.1 -- 253MB
mcr.microsoft.com/dotnet/core/runtime:2.1 -- 180MB
```

> For more information please see the Docker Documentation for [Multi-Stage Builds](TODO).

## Agents

When a pipeline is triggered, Azure DevOps initiates a build by adding the build job to an agent queue. These queues container build agents, which are the actually environments that will execute the steps defined in the pipeline. 

One or more agents may be added into a pool. Using a pool allows multiple agents to be used in parallel, decreasing the time that a job sits in the queue awaiting an available agent. Microsoft maintains "hosted" agents, or you can define and run your own "self-hosted" within a physical server, virtual machine, or even container.

### Hosted Agents

The easiest way to execute builds on Azure DevOps is by using the hosted build agents. These VM-based agent pools come in a variety of operating systems, and have Docker available. Pipeline steps such as `docker build` are available without additional configuration.  

Being a hosted environment, there is minimal ability to customize the SDKs, tooling, and other components within the virtual machine. A pipeline step could be used to install needed software via approaches such as `apt-get` or `chocolatey`, however doing so may add substantial time to each build considering the VMs are wiped after the pipeline completes.

The use of multi-stage builds decreases this dependency on the hosted build environment, as all necessary components for a container image should be embedded into the "builder" Dockerfile stage. To adjust the container runtime itself, for example to use a supported Docker Enterprise engine, a self-hosted agent is necessary.

### Self-Hosted Agents

Running a self-hosted agent provides the ability to customize every facet of the build environment. Container runtimes can be customized, specific SDKs and compilers added, and for scenarios where project requirements restrict cloud-based agents a self-hosted agent can enable on-premises builds.

The downsize is that infrastructure must be deployed and maintained to host the agents. These severs or virtual machines install a software agent, which connects to a specific Azure DevOps tenant. 

The software agent can also run within a Docker container. Deploying containerized agents provides an array of benefits compared to traditional VM-based build agents including:

* Utilize existing container clusters, such as Docker Enterprise, rather than dedicated infrastructure

* Easily scale up and down the number of build agent instances with the Kubernets or Swarm orchestrators

* Minimal assets to maintain when multi-stage builds are used as the containerized agent only needs access to the host's Docker daemon

If bulding traditional software that does not run in a container, then simply install the Azure DevOps agent and connect to a tenant. For containerized applications, the Docker CLI is installed within the container so that it may connect to the host's Docker Daemon. To make this connection on Linux, a Docker Volume is used to mount the host's daemon:

```bash
docker run --volume /var/run/docker.sock:/var/run/docker.sock ...
```

And on Windows a Named Pipe is monuted via a similar approach:

```powershell
docker run --volume \\.\pipe\docker_engine:\\.\pipe\docker_engine ...
```

These volume mounts allow the Docker CLI to execute commands such as `docker build` within the container, and have the action executed on the host.

#### Linux Container Build Agent

Microsoft [provides documentation](https://docs.microsoft.com/en-us/azure/devops/pipelines/agents/docker?view=azure-devops#linux) on running a build agent within a container. The base instructions may be extended as-needed, for example to add binaries such as `kubectl` and `helm`:

```dockerfile
#=========================================================
# Stage 1: Download Docker CLI binary
#=========================================================
FROM alpine:latest AS dockercli

ARG DOCKER_BRANCH=test
ARG DOCKER_VERSION=19.03.0-beta4

RUN wget --output docker.tgz https://download.docker.com/linux/static/${DOCKER_BRANCH}/x86_64/docker-${DOCKER_VERSION}.tgz && \
    tar -zxvf docker.tgz && \
    chmod +x docker/docker

#=========================================================
# Stage 2: Download kubectl binary 
#=========================================================
FROM alpine:latest AS kubectl

ARG KUBECTL_VERSION=v1.14.1

RUN wget --output ./kubectl https://storage.googleapis.com/kubernetes-release/release/${KUBECTL_VERSION}/bin/linux/amd64/kubectl && \
    chmod +x ./kubectl

#=========================================================
# Stage 3: Download Helm binary 
#=========================================================
FROM alpine:latest AS helm

ARG HELM_VERSION=v2.13.1

RUN wget --output helm.tar.gz https://storage.googleapis.com/kubernetes-helm/helm-${HELM_VERSION}-linux-amd64.tar.gz && \
    tar -zxvf helm.tar.gz && \
    chmod +x linux-amd64/helm

#=========================================================
# Stage 4: Setup Azure Pipelines remote agent 
# Documented at https://docs.microsoft.com/en-us/azure/devops/pipelines/agents/docker?view=azure-devops#linux
#=========================================================
FROM ubuntu:16.04

# To make it easier for build and release pipelines to run apt-get,
# configure apt to not require confirmation (assume the -y argument by default)
ENV DEBIAN_FRONTEND=noninteractive
RUN echo "APT::Get::Assume-Yes \"true\";" > /etc/apt/apt.conf.d/90assumeyes

RUN apt-get update \
&& apt-get install -y --no-install-recommends \
        ca-certificates \
        curl \
        jq \
        git \
        iputils-ping \
        libcurl3 \
        libicu55 \
        zip \
        unzip

WORKDIR /azp

COPY ./start.sh .
RUN chmod +x start.sh

# User is required for the Docker Socket
USER root 

# Copy binaries from earlier build stages
COPY --from=dockercli docker/docker /usr/local/bin/docker
COPY --from=kubectl ./kubectl /usr/local/bin/kubectl
COPY --from=helm /linux-amd64/helm /usr/local/bin/helm

CMD ["./start.sh"]
```

#### Windows Container Build Agent
TODO: make better
Microsoft does not maintain specific instructions for running the cross-platform build agent software within a Windows Container, but a similar approach to the Linux Container build agent may be taken: 

```dockerfile
# escape=`
FROM mcr.microsoft.com/windows/servercore:ltsc2019

SHELL ["powershell", "-Command", "$ErrorActionPreference = 'Stop'; $ProgressPreference = 'SilentlyContinue';"]

ENV AGENT_URL='https://vstsagentpackage.azureedge.net/agent/2.148.2/vsts-agent-win-x64-2.148.2.zip'
ENV DOCKER_VERSION='18.09'

WORKDIR C:\agent

# Setup Docker CLI
RUN Invoke-WebRequest `
      -UseBasicParsing `
      -OutFile docker.zip `
      -Uri https://download.docker.com/components/engine/windows-server/18.09/docker-18.09.6.zip; `
    Expand-Archive docker.zip -DestinationPath $Env:ProgramFiles -Force; `
    $env:Path = 'C:\ProgramFiles\docker;' + $env:PATH; `
    Remove-Item docker.zip;

# Setup general tools
RUN Invoke-Expression (New-Object Net.WebClient).DownloadString('https://get.scoop.sh'); `
  scoop install git;

# Setup Azure Pipelines Agent
RUN Invoke-WebRequest `
      -OutFile C:\agent\agent.zip `
      -Uri $env:AGENT_URL `
      -UseBasicParsing; `
    Expand-Archive `
      -Path agent.zip `
      -Destination C:\agent; `
    Remove-Item agent.zip;

# Copy startup script into container
COPY start.ps1 .

# Run startup script on initialization
CMD .\start.ps1
```

PowerShell used as the `CMD .\start.ps1` command:

```powershell
# Check if required variables are present
if (!$env:AZP_URL) { Write-Host "The AZP_URL environment variable is null. Please adjust before continuing"; exit 1; }
if (!$env:AZP_TOKEN) { Write-Host "The AZP_TOKEN environment variable is null. Please adjust before continuing"; exit 1; }
if (!$env:AZP_POOL) { $env:AZP_POOL='Default' }

# ===============================
# Configure Azure Pipelines Agent
# ===============================
.\config.cmd `
  --unattended `
  --url "${env:AZP_URL}" `
  --auth PAT `
  --token "${env:AZP_TOKEN}" `
  --pool "${env:AZP_POOL}" `
  --replace `
  --acceptTeeEula

# ==============================
# Run Azure Pipelines Agent
# ==============================
.\run.cmd
```

> Use Docker Secrets rather than an environment varibale for the Azure DevOps Personal Access Token (PAT) for increased security

## Signing Images with Docker Content Trust

### Using the Azure Pipelines Library

## Deploying 

### Deploying with Helm

### Deploying with Docker App




- Initiate build from DTR via PAT & REST when new image is mirrored.