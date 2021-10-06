# Creating and Running Containers

Applications ultimately are all comprised of one or more programs that run on individual machines.

Applications are comprosed of a language runtime, libraries, and your source code. This dependency on shared libraries causes problems when an application developed on a programmer's laptop has a dependency on a shared library that isn't available when the program is rolled out to the production OS.

A program can only execute successfully if it can be reliably deployed onto the machine it should run.

When working with applications it's often helpful to package them in a way that makes it easy to share them with others. Docker, the default container runtime engine, makes it easty to package an executable and push it to a remote registry where it can later be pulled by others.

Container images bundle a program and its dependencies into a single artifcat under a root filesystem.

## Container Images

A _container image_ is a binary package that encapsulates all of the files necessary to run a program inside of an OS container. Depending on how your first experiment with containers, you will either build a container image from your local filesystem or download a preexisting image from a _container registry_.

Container images are typically combined with a container configuration file, which provides instructions on how to set up the container environment and execute an application entry point.

### The Docker Image Format

The most popular image format is the Docker format, that is ran using the `docker` command.

Docker images are made up of a series of filesystem layers.

Each layer adds, removes, or modifies files from the preceding layer in the filesystem.


#### Container Layering

Container images are constructed with a series of filesystem layers, where each layer inherits and modifies the layers that came before it.

Conceptually, each container image layer builds upon a previous one. Each parent reference is a pointer.

## Building Application Images with Docker

### Dockerfiles

A Dockerfile can be used to automate the creation of a Docker container image.

```
# Start from a Node.js 10 (LTS) image
FROM node:10

# Specify the directory inside the image in which all commands will run
WORKDIR /usr/src/app

# Copy package files and install dependencies
COPY package*.json ./
RUN npm install

# Copy all of the app files into the image
COPY ..

# The default command to run when starting the container
CMD [ "npm", "start" ]

```


### Optimizing Image Sizes


There are several gotchas that come when people begin to experiment with container images that lead to overly large images. The first thing to remember is that files that are removed by subsequent layers in the system are actually still present in the images; they're just inaccessible.

* For instance:
    * Layer A: contains a large file named 'BigFile'
        * Layer B: removes 'BigFile'
            * Layer C: builds on B by adding a static binary
    * _BigFile_ is still transmitted through the network, even if you can no longer access it.

You might think that BigFile is no longer present in this image. After all, when you run the image, it is no longer accessible. But in fact it is still present in layer A, which means that whenever you push or pull the image, BigFile is still transmitted through the network, even if you can no longer access it.

In general, yo want to order your layers from least likely to change to most likely to change in order to optimize the image size for pushing and pulling.

### Image Security

Do not build containers with passwords baked in - and this includes not just in the final layer, but any layers in the image. One of the counterintuitive problems introduced by container layers is that deleting a file in one layer does not delete that file from preceding layers.

Secrets and images should _never_ be mixed.

## Multistage Image Builds

One of the most common ways to accidentally build large images is to do the actual program compilation as part of the construction of the application container image.

The trouble with doing this is that it leaves all of the unnecessary development tools, which are usually quite large, lying around inside of your image and slowing down your deployments.

To resolve this problem, Docker introduced _multistage builds_.


```

FROM golang:1.11-alpine

# Install Node and NPM
RUN apk update && apk upgrade && apk add --no-cache git nodejs bash npm

# Get dependencies for Go part of build
RUN go get -u github.com/jteeuwen/go-bindata/...
RUN go get github.com/tools/godep

WORKDIR /go/src/github.com/kubernetes-up-and-running/kuard

# Copy all sources in
COPY . .

# This is a set of variables that the build script expects
ENV VERBOSE=0
ENV PKG=github.com/kubernetes-up-and-running/kuard
ENV ARCH=amd64
ENV VERSION=test

# Do the build. This script is part of incoming sources.
RUN build/build.sh
CMD [ "/go/bin/kuard" ]

```

In the image above, Go development tools and the tools to build the React.js frontend and the source code for the application are not needed by the final application. The image, across all layers, adds up to over 500 MB.

With _multistage builds_, only the necessary is added to the final image, as shown below:

```

# STAGE 1: Build
FROM golang:1.11-alpine AS build

# Install Node and NPM
RUN apk update && apk upgrade && apk add --no-cache git nodejs bash npm

# Get dependencies for Go part of build
RUN go get -u github.com/jteeuwen/go-bindata/...
RUN go get github.com/tools/godep

WORKDIR /go/src/github.com/kubernetes-up-and-running/kuard

# Copy all sources in
COPY . .

# This is a set of variables that the build script expects
ENV VERBOSE=0
ENV PKG=github.com/kubernetes-up-and-running/kuard
ENV ARCH=amd64
ENV VERSION=test

# Do the build. Script is part of incoming sources.
RUN build/build.sh

# STAGE 2: Deployment

FROM alpine

USER nobody:nobody

COPY --from=build /go/bin/kuard /kuard

CMD [ "/kuard" ]

```

This Dockerfile produces two images. The first is the build image, which contains the Go compiler, React.js toolchain, and source code for the program. The second is the deployment image, which simply contains the compiled binary. The final image produced from this Dockerfile is somewhere around 20 MB.


## Storing Images in a Remote Registry

What good is a container image if it's only available on a single machine?

The standard within the Docker community is to store Docker images in a remote registry.

There are private and public registries.

The most popular is the Docker Hub image registry.


## The Docker Container Runtime

### Running Containers with Docker

The Docker CLI tool can be used to deploy containers: 

`docker run -d --name kuard --publish 8080:8080 gcr.io/kuar-demo/kuard-amd64:blue`

### Exploring he kuard Application

`curl http://localhost:8080`


### Limiting Resource Usage

This allows multiple applications to coexist on the same hardware and ensures fair usage.

#### Limiting memory resources

```
docker run -d 
    --name kuard 
    --publish 8080:8080
    --memory 200m // limits kuard to 200 MB
    --memory-swap 1G // limits kuard to 1 GB of swap
    --cpu-shares 1024  // limites the CPU utilization
    gcr.io/kuar-demo/kuard-amd64:blue

```