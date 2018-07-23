# Installing Docker Community Edition (CE) and Docker Compose on Ubuntu 18.04 LTS
This tutorial will go through the steps of setting up Docker and Docker Compose on Ubuntu 18.04 LTS as it will be used for the XeAP2 lab environment.


## Remove Ubuntu apt Docker version 

Remove the Docker version that comes from the Ubuntu apt package manager as it is an older version which does not come with Docker Compose. 

    $ sudo apt-get remove docker docker-engine docker.io

## Setup the Docker Repository

Update apt packages to ensure we receive the most up-to-date packages when installing from the apt package manager.

    $ sudo apt-get update
   
Install pre-requisite software needed to install Docker from the Docker repo.

    $ sudo apt-get install \
      apt-transport-https \
      ca-certificates \
      curl \
      software-properties-common
      
Retrieve the official Docker repository GPG key add it to apt.

    $ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
    
Verify the key by using the following fingerprint: ```9DC8 5822 9FC7 DD38 854A E2D8 8D81 803C 0EBF CD88```


  ```
  $ sudo apt-key fingerprint 0EBFCD88
  ```
  
  ```
  pub   4096R/0EBFCD88 2017-02-22
        Key fingerprint = 9DC8 5822 9FC7 DD38 854A  E2D8 8D81 803C 0EBF CD88
  uid                  Docker Release (CE deb) <docker@docker.com>
  sub   4096R/F273FCD8 2017-02-22
  ```
  
Add the stable Docker repository to the apt package manager

    $ sudo add-apt-repository \
       "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
       $(lsb_release -cs) \
       stable"

## Install Docker CE

Update the Docker CE repository within the apt package manager

    $ sudo apt-get update
    
Install the latest version of Docker CE

    $ sudo apt-get install docker-ce
    
Verify the Docker CE installation by running the ```hello-world``` image. Docker will download the image and run it within a container.

  ```
  $ sudo docker run hello-world
  ```

  ```
  Unable to find image 'hello-world:latest' locally
  latest: Pulling from library/hello-world
  9db2ca6ccae0: Pull complete
  Digest: sha256:4b8ff392a12ed9ea17784bd3c9a8b1fa3299cac44aca35a85c90c5e3c7afacdc
  Status: Downloaded newer image for hello-world:latest

  Hello from Docker!
  This message shows that your installation appears to be working correctly.

  To generate this message, Docker took the following steps:
   1. The Docker client contacted the Docker daemon.
   2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
      (amd64)
   3. The Docker daemon created a new container from that image which runs the
      executable that produces the output you are currently reading.
   4. The Docker daemon streamed that output to the Docker client, which sent it
      to your terminal.

  To try something more ambitious, you can run an Ubuntu container with:
   $ docker run -it ubuntu bash

  Share images, automate workflows, and more with a free Docker ID:
   https://hub.docker.com/

  For more examples and ideas, visit:
   https://docs.docker.com/engine/userguide/
  ```
  
Enable Docker to start on boot

    $ sudo systemctl enable docker
    

## Docker Compose Overview

Docker Compose is an addon to Docker which allows the user to define and setup multi-container Docker applications using a YAML configuration file. Docker Compose will be used for the XeAP2 Ancillary Tools session.

## Docker Compose Installation

Download the latest version of Docker Compose binary:

    $ sudo curl -L https://github.com/docker/compose/releases/download/1.21.2/docker-compose-$(uname -s)-$(uname -m) -o /usr/local/bin/docker-compose

Give executable permissions to the Docker Compose binary:

    $ sudo chmod +x /usr/local/bin/docker-compose
    
Verify the installation

```
$ docker-compose --version
```

```
docker-compose version 1.21.2, build 1719ceb
```


### References    
Docker CE Ubuntu Installation: https://docs.docker.com/install/linux/docker-ce/ubuntu/#set-up-the-repository
Docker Compose Installation: https://docs.docker.com/compose/install/

