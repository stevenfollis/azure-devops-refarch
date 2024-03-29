---
title: Automation with Azure DevOps Reference Architecture for Docker Enterprise Edition
summary: This Solution Brief documents how to integrate Azure DevOps with Docker Enterprise to enable continuous integration and continuous delivery practices
type: refarch
author: stevenfollis
product:
- ee
testedon:
- EE 2.0
- EE 2.1
- ee-17.06.2-ee-17
- ee-18.09.00-ee
- ucp-3.0.6
- ucp-3.1.6
- dtr-2.6.0
- dtr-2.6.3
platform:
- linux
- windows
tags:
- devops
- azure
- ci
- cd
---

# Automating Docker Enterprise with Azure DevOps

## Introduction

Microsoft Azure DevOps (ADO) is a fully managed suite of tooling that empowers developers and operators to implement DevOps techniques. The [Docker Enterprise Platform](https://www.docker.com/products/docker-enterprise) integrates with ADO through the use of Azure Pipelines for automated building and deploying of container-based workloads. 

This guide does not attempt to recreate the excellent [Azure DevOps documentation](https://docs.microsoft.com/en-us/azure/devops) available online, but will focus on integration points between the two systems.

## Setting up Azure DevOps

An Azure DevOps tenant is necessary to use the service. These accounts are available from a myriad of different sources from Microsoft Developer Network (MSDN) subscriptions, to simply [signing up](https://azure.microsoft.com/en-us/services/devops/) for a free account.


### Creating a Project

Once a tenant is secured, [create an Organization](https://docs.microsoft.com/en-us/azure/devops/organizations/accounts/create-organization) and a Project to hold code and pipeline definitions.

### Configuring Source Control Management

Azure DevOps is capable of working with git repositories hosted in ADO itself, or in a variety of other locations such as GitHub, or on-premises servers. Create a new git repository in the ADO Project, or link to an existing repository to begin the enablement of build automation.

Automated triggers can be used between the git repository and ADO, meaning that when a `git commit push` occurs then an ADO Pipeline can be automatically initiate. This effectively established a continuous integration (CI) capability for source code.

Once the git repository is linked with ADO, ensure that all application code has been committed.

## Building a Pipeline

[Pipelines](https://docs.microsoft.com/en-us/azure/devops/pipelines/get-started/overview?view=azure-devops) in Azure DevOps define a series of steps the sequentially build, test, and package applications in various forms. 

Pipelines can be generated via two techniques. The Classic experience is a GUI-driven wizard where boxes and dropdowns are completed with pipeline steps. This system has been largely replaced with a YAML-based system more in line with other offerings in the DevOps market. 

The YAML-based pipelines offer "pipelines as code" benefits, as they are committed to source control and able to versioned and shared like any other file. This guide will focus on YAML-based pipelines.

### Triggers

Pipelines are initiated via [Triggers](https://docs.microsoft.com/en-us/azure/devops/pipelines/build/triggers), which contain the logic that determines how and when a pipeline beings execution. Triggers can be configured in a variety of ways:

* **Manual** will require that an operator initiate a new build

* **Automatic** will initiate a build any time that a commit is added to the designated source control system. To avoid builds when non-application changes are made to the repository, specifcy a path and/or branch to specifically target application code changes.

  ```yaml
  trigger:
    branches:
      include:
      - master
    paths:
      include:
      - app/src/*
  ```

### Pools

When a build is triggered it is added to a queue for a given [agent pool](https://docs.microsoft.com/en-us/azure/devops/pipelines/agents/pools-queues). The next available agent will then have the build assigned, and it will execute the pipeline steps.

By default ADO uses a [hosted agent pool](https://docs.microsoft.com/en-us/azure/devops/pipelines/agents/hosted?view=azure-devops) where all servers are maintained by Microsoft. Alternatively, a pool of [custom agents](https://docs.microsoft.com/en-us/azure/devops/pipelines/agents/v2-linux?view=azure-devops) may also be used. Please see the Agents section for more detailed information on build agent setup.

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

The steps within a pipeline define which actions are done to the source code. These actions may be defined as custom scripts or configured via pre-built tasks.

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

Build a Docker Image with a PowerShell script:

```yaml
scripts:
- powershell: |
    docker build `
      --tag $(DOCKER_REGISTRY_IMAGE):"$(git rev-parse --short HEAD)" `
      .\moby\pipeline
  displayName: 'Build Docker Image'
```

Tasks are pre-built actions that can easily be integrated into a pipeline. A series of tasks are available out of the box from Microsoft; however the system is also extensible through the Visual Studio Marketplace community. 

> Pre-built tasks for Docker and Kubernetes are available, however the typical brevity of the `docker` and `kubectl` command lines make the additional abstraction optional compared to use of simple `script` tasks

### Variables

Dockerfiles are often static assets, requiring a developer to commit code changes to adjust its behavior. Hard-coded values also impede the ability to reuse a Dockerfile across multiple contexts or image variations. The Azure DevOps platform offers variables to be defined for a given Pipeline, radically increasing the flexibility of Dockerfiles by not requiring code changes to reuse a given file. 

While variables may be named any value, it is recommended to decide upon a naming convention that promotes consistency and predictability. `DOCKER_` is one such convention prefix that clearly denotes that a variable is related to Docker. For example, `DOCKER_REGISTRY_USERNAME` would denote first that a value is related to Docker, that it is used to interact with a Registry, and that it contains an account username.

Values that may contain sensitive or secret information can make use of a Build Secret, rather than a Build Variable. When creating a variable, simply select the lock icon to convert the value to a secret. Secrets are not echoed out in logs or allowed to be seen once set. Setting the password or token used to authenticate with a Docker Registry via a `DOCKER_REGISTRY_TOKEN` secret would be advisable instead of a variable.

### Docker Enterprise Service Account

Steps within an Azure DevOps Pipeline that require interaction with Docker Enterprise may use a service account model for clean separation between systems. In Universal Control Plane, a new user account may be created with a name such as `azure-devops` or similar that will serve as a service account. If using [LDAP](https://docs.docker.com/ee/ucp/admin/configure/external-auth/) or [SAML](https://docs.docker.com/ee/ucp/admin/configure/enable-saml-authentication/) integration with a directory such as Active Directory then create an account in the external system to be synchronized into UCP.

![service account](./images/service-account.png)

This service account is then used whenever a pipeline needs to interact with Docker Enterprise. For example, to execute a `docker push` into Docker Trusted Registry, the pipeline must first authenticate against the registry with a `docker login`:

```yaml
- script: |
    docker login $(DOCKER_REGISTRY_FQDN) \
      --username $(DOCKER_REGISTRY_USERNAME) \
      --password $(DOCKER_REGISTRY_TOKEN)
  displayName: 'Login to Docker Trusted Registry'
```

In this example the `DOCKER_REGISTRY_USERNAME` refers to the service account's username, and the `DOCKER_REGISTRY_TOKEN` is an Access Token [generated from DTR](https://docs.docker.com/ee/dtr/user/access-tokens/) loaded into Azure DevOps as a Secret.

User accounts in Docker Enterprise utilize granular, [role-based access controls (RBAC)](https://docs.docker.com/ee/ucp/authorization/) to ensure that only the proper account has access to a given DTR repository, set of UCP nodes, etc. The service account can be directly granted permissions for pertinent DTR repositories or added to a UCP Group that inherits permissions. This system ensures that the service account has the least privileges necessary to conduct its tasks with Docker Enterprise.

A Docker [Client Bundle](https://docs.docker.com/ee/ucp/user-access/cli/#download-client-certificates) can also be generated for this account, which can be used for continuous delivery tasks such as `docker stack deploy` or `helm upgrade`.

## Preparing a Dockerfile

A developer working with a Dockerfile in their local environment has different requirements than a build automation system using the same file. A series of adjustments can optimize a Dockerfile for build performance and enhance the flexibility of a file to be utilized in multiple build variations.

### Build Arguments

The mechanism to dynamically pass a value into a Dockerfile at `docker build` time is the `--build-arg` flag. A variable or secret can be used with the flag to change a build outcome without committing a code change into the source control system. To utilize the flag, we add an `ARG` line to our Dockerfile for each variable to be passed. 

For example, to dynamically expose a port with in the Dockerfile we would adjust:

```dockerfile
FROM mcr.microsoft.com/dotnet/core/sdk:2.1
EXPOSE 80
WORKDIR /app
```

to include an `ARG`

```dockerfile
FROM mcr.microsoft.com/dotnet/core/sdk:2.1
ARG DOCKER_IMAGE_PORT=80
EXPOSE ${DOCKER_IMAGE_PORT}
WORKDIR /app
```

Note that we set a default value by including `=80`; this value will be used if a dynamic value is not passed in at build time.

A base image may also be made into a dynamic value, however the `ARG` must be placed outside of the `FROM` statement. For example, to adjust the following Dockerfile:

```dockerfile
FROM mcr.microsoft.com/dotnet/core/sdk:2.1
EXPOSE 80
WORKDIR /app
```

Place the `ARG` at the beginning of the file and use the variable name in the `FROM` statement:

```dockerfile
ARG BASE_IMAGE='mcr.microsoft.com/dotnet/core/sdk:2.1'

FROM ${BASE_IMAGE}
EXPOSE 80
WORKDIR /app
```

Positioning the `ARG` outside of the `FROM` statement(s) places it in a higher scope than within any specific stage. 

Using the `ARG` and `--build-arg` pattern is useful to easily patch a given image when improvements are made to its base image. Adjusting a build variable and initiating a build brings the newer base image tag in without requiring a formal code commit.

### Labels

Metadata may be added to a Dockerfile via the `LABEL` keyword. Labels can help designate particular application owners, points of contact, or extraneous characteristics that would benefit from embedding within an image. 

Some common settings in Dockerfiles include:

```dockerfile
FROM mcr.microsoft.com/dotnet/core/sdk:2.1
LABEL IMAGE_OWNER='Moby Whale <mobyw@docker.com>'
LABEL IMAGE_DEPARTMENT='Digital Marketing'
LABEL IMAGE_BUILT_BY='Azure DevOps'
EXPOSE 80
WORKDIR /app
```

Combine `LABEL` with `ARG` to dynamically pass metadata values into an image at build time.

```dockerfile
FROM mcr.microsoft.com/dotnet/core/sdk:2.1
ARG COMMIT_ID=''
LABEL GIT_COMMIT_ID=${COMMIT_ID}
```

```powershell
docker build `
  --build-arg COMMIT_ID="$(git rev-parse --short HEAD)" `
  --tag $(DOCKER_REGISTRY_IMAGE):"$(git rev-parse --short HEAD)" `
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

Dockerfiles originally functioned as a single "stage", where all steps took place in the same context. All libraries and frameworks necessary for the Dockerfile to build had to be loaded in, bloating the size of the resulting images. Much of this image size was used during the build phase, but was not necessary for the application to properly run; for example after an application compiles it does not necessarily have to have the compiler or SDK within the image to run. 

The introduction of "[multi-stage builds](https://docs.docker.com/develop/develop-images/multistage-build/)" introduced splitting out of individual "stages" within one physical Dockerfile. In a build system such as Azure DevOps, we can define a "builder" stage with a base layer containing all necessary compilation components, and then a second, lightweight "runtime" stage devoid of hefty SDKs and compilation tooling. This last stage becomes the built image, with the builder stage serving only as a temporary intermediary. 

```dockerfile
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

[Docker Content Trust (DCT)](https://docs.docker.com/engine/security/trust/content_trust/) is a security feature built into Docker Enterprise that enables Docker container images to be cryptographically signed. These signatures provide assurances that a given image was built within an organization, rather than sourced elsewhere. While a signed image is useful, a unique feature of Docker Enterprise is the ability to enforce policies within Universal Control Plane (UCP) that require only signed images to be scheduled onto the cluster.

Container images may be signed by individual developers, or as a step within the continuous integration process. Azure DevOps Pipelines can execute image signing, typically with a service account created in Docker Enterprise. 

Before integrating a Pipeline with DCT, ensure that pertinent repositories in Docker Trusted Registry have been initialized for DCT, and that the service account to be used has been delegated signing permissions.

### Using the Azure Pipelines Library

Docker Content Trust works with a series of cryptographic certificates, and each user has a set of such certificate keys issued in a [Client Bundle](https://docs.docker.com/ee/ucp/user-access/cli/). Azure DevOps requires these keys to execute image signing, but the certificates are sensitive values that need protection. 

Sensitive strings may be loaded into Azure DevOps' [Secrets](https://docs.microsoft.com/en-us/azure/devops/pipelines/process/variables?view=azure-devops#secret-variables) feature, however the Client Bundle is a zipped folder of certificates. To handle this sensitive file, the zipped Client Bundle file may be uploaded into the Azure DevOps [Secure File Library](https://docs.microsoft.com/en-us/azure/devops/pipelines/library/secure-files).

![secure file library](./images/secure-file-library.png)

When a Client Bundle needs to be revoked or cycled, delete the uploaded bundle from the library and upload a new file with fresh certificates. If the filename is identical then the Azure DevOps Pipeline does not need to be adjusted - it will use the new file.

### Signing an Image

The Secure File Library allows Azure DevOps to handle the security of the file, including a pre-built Pipeline task that enables easy transfer of the file into a build agent during Pipeline bulds. Pipeline authors do not have to handle the downloading and security handshake to retrieve the file, which simplifies the Pipeline logic. 

Once the client bundle is downloaded, a step is needed to unzip the file (ensure that any self-hosted agents have the `unzip` package or similar installed). Once the files are unzipped, the `docker trust key load` command is used to load the service account's key into the Docker Engine. 

Finally, the Docker Engine is set to use Docker Content Trust. In this example the configuration option is set with the `DOCKER_CONTENT_TRUST` variable (defined here inline but may be at a higher scoped level such as a `jobs` block) but alternatively this may be set [in the Docker Enterprise Engine configuration file](https://docs.docker.com/engine/security/trust/content_trust/#enabling-dct-within-the-docker-enterprise-engine).

```yaml
- task: DownloadSecureFile@1
    inputs:
      secureFile: 'ucp-bundle-azure-devops.zip'

- script: |
    # Unzip Docker Client Bundle from UCP
    unzip \
      -d $(Agent.TempDirectory)/bundle \
      $(Agent.TempDirectory)/ucp-bundle-azure-devops.zip
    
    # Load key for use with Docker Content Trust (DCT)
    docker trust key load $(Agent.TempDirectory)/bundle/key.pem
  displayName: 'Setup Docker Client Bundle'

- script: |
    # Push image to Docker Trusted Registry and sign with DCT
    # Makes use of environment variables
    DOCKER_CONTENT_TRUST=1 \
    docker push $(DOCKER_REGISTRY_IMAGE):"$(git rev-parse --short HEAD)"
  displayName: 'Push Docker Image'
```

During the `docker push` the image will be uploaded to Docker Trusted Registry (DTR), and then signed with the key loaded into the Engine. The **Signed** column in DTR will now reflect that a given tag is signed, and more information may be retrieved via the `docker trust inspect` command.

![signed dtr images](./images/dtr-tags-signed.png)

```json
$ docker trust inspect dtr.moby.org/se-stevenfollis/moby:3a26dd5
[
    {
        "Name": "dtr.moby.org/se-stevenfollis/moby:3a26dd5",
        "SignedTags": [
            {
                "SignedTag": "3a26dd5",
                "Digest": "152c6e774c2d39df8da4f6901b4ee4979efe39fc25688cba24c4d39e019b246e",
                "Signers": [
                    "azuredevops"
                ]
            }
        ],
        "Signers": [
            {
                "Name": "stevenfollis",
                "Keys": [
                    {
                        "ID": "a7d7f4d91b2489fe0a7561985401bd6959a925bbbbcfa822a257d8601dd4272b"
                    }
                ]
            },
            {
                "Name": "azuredevops",
                "Keys": [
                    {
                        "ID": "8d947f076173d5972095159eeedd5b0ab691159641b01e59214d8d943db04e1d"
                    }
                ]
            }
        ],
        "AdministrativeKeys": [
            {
                "Name": "Root",
                "Keys": [
                    {
                        "ID": "070edd256b81b1c046fe197b8e409999e86d8c587d36008a7f4db62501117623"
                    }
                ]
            },
            {
                "Name": "Repository",
                "Keys": [
                    {
                        "ID": "ef6dc1c0860ef0bb7157f8c4ec8141398940757bd2ba58654cc6ade978ae1ec1"
                    }
                ]
            }
        ]
    }
]
```

## Deploying 

Azure DevOps Pipelines may be used for both continuous integration, and for continuous delivery processes. In the integration stage a Docker image was built and pushed into Docker Trusted Registry (DTR). Delivery takes the next step to schedule the image from DTR onto a cluster of servers managed by Universal Control Plane.

### Deploying with Helm

In the Kubernetes world, [Helm](https://helm.sh/) is a popular tool for deploying and managing the life cycle of container workloads. A Helm "Chart" is created via the `helm create` command, and then values are adjusted to match a given application's needs. Docker Enterprise is a CNCF Certified distribution of Kubernetes and works seamlessly with Helm.

Azure DevOps Pipelines can interface with Docker Enterprise via Helm by having the `kubectl` binary installed in the build agent. This command line tool is then further configured to work with UCP through a Docker Client Bundle, which establishes a secure connection context between `kubectl` and UCP. Once established, standard Helm commands may be issued to update a running Helm workload. 

In the following example, a formal `deploy` [stage](https://docs.microsoft.com/en-us/azure/devops/pipelines/process/stages) is created in the Azure DevOps Pipeline that depends on the successful completion of a `build` stage. A Client Bundle is then downloaded from the Azure DevOps [Secure File Library](https://docs.microsoft.com/en-us/azure/devops/pipelines/library/secure-files), unzipped, and sourced. The `helm upgrade` command then updates the declared image tag to the recently build tag from DTR with `--set "image.tag=$(git rev-parse --short HEAD)"`. Helm then gracefully handles the upgrade process for the running image.

```yaml
- stage: deploy
  displayName: Deploy to Cluster
  dependsOn:
  - build
  jobs:
  - job: helm
    displayName: 'Deploy Container with Helm'
    pool:
      name: 'Shared SE - Linux'
      demands:
      - agent.os -equals Linux
      - docker
    steps:
    - task: DownloadSecureFile@1
      inputs:
        secureFile: 'ucp-bundle-azure-devops.zip'

    - script: |
        # Unzip Docker Client Bundle from UCP
        unzip \
          -d $(Agent.TempDirectory)/bundle \
          $(Agent.TempDirectory)/ucp-bundle-azure-devops.zip
      displayName: 'Setup Docker Client Bundle'

    - script: |
        # Connect Helm to cluster via Docker Client Bundle
        cd $(Agent.TempDirectory)/bundle
        eval "$(<env.sh)"

        # Update deployment with Helm
        cd $(Build.SourcesDirectory)
        helm upgrade \
          web \
          ./pipelines/helm/techorama \
          --set "image.tag=$(git rev-parse --short HEAD)" \
          --tiller-namespace se-stevenfollis
      displayName: 'Update application with Helm'
```


TODO
- Initiate build from DTR via PAT & REST when new image is mirrored.

- ToC

- Docker Context