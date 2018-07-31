# National RADIUS Server (NRS) Docker Container

The NRS Docker container uses radsecproxy on a base Ubuntu 18.04 LTS image. 

Docker Hub: https://hub.docker.com/r/spgreen/eduroam-radsecproxy/

## Using the NRS Docker Image

Create or use existing a radsecproxy configuration file in your desired directory; it will be mounted into the container on start-up.

Create a log file if you want logs to be saved on the Docker host machine: E.g. ```# touch /var/log/radsecproxy.log```

### Foreground Start
Start the container in the foreground using the following command. Remember to change the file paths for the radsecproxy log and configuration file.

    $ sudo docker run -it --name nrs \
      -p 1812:1812/udp \
      -p 1813:1813/udp \
      -e TZ=Pacific/Auckland \
      -e ENVIRONMENT=TEST \
      -v /path/to/log/file.log:/var/log/radsecproxy/radsecproxy.log \ 
      -v /path/to/radsecproxy.conf:/etc/radsecproxy.conf \
      spgreen/eduroam-radsecproxy:1.7.1-xeap2
   
**Flag Explanation**   
- `-p 1812:1812/udp`:
    - Opens UDP port 1812 on the host machine and maps it to UDP port 1812 on the Docker container. Therefore if the host machine receives a RADIUS request on UDP port 1812, the host will send said request to the Docker container for processing. 
- `-e TZ=Pacific/Auckland`:
    - TZ is an environment variable that sets the timezone of the container; change to your desired timezone. Default: UTC
- `-e ENVIRONMENT=TEST`:
    - If the ENVIRONMENT environment variable is set to TEST, the Docker container will send logs to stdout. Used only for initial debugging and should not be used in a production envrionment as logs will be saved to a file.
- `-v /path/to/log/file:/var/log/radsecproxy/radsecproxy.log`:
    - Mounts host machine file at `/path/to/log/file.log` to `/var/log/radsecproxy/radsecproxy.log`. Provides the ability to save files to the host machine or place configuration files within the Docker container before the container starts.
    
 **Please change the log file (`/path/to/log/file.log`) and configuration file (`/path/to/radsecproxy.conf`) paths for the above example.**
    
If the configuration is correct, you will see a similar output:

    Jul 23 07:07:46 2018: createlistener: listening for udp on *:1812
    
The radsecproxy NRS Docker container will not start if there is an issue with the radsecproxy configuration file.

**The NRS is ready and now listening for RADIUS requests**

### Background Start

Once you know that the NRS Docker works, you would want to the container in the backgroud. This can be done by using the ```-d``` flag within the ```docker run``` command.

Stop and remove the current NRS Docker container. Since we used a name to initialise the container, we can stop and remove it using that same name. 

    $ sudo docker stop nrs; sudo docker rm nrs

Restart the container but this time in background mode

    $ sudo docker run -d --name nrs \
      -p 1812:1812/udp \
      -p 1813:1813/udp \
      -e TZ=Pacific/Auckland \
      -v /path/to/log/file.log:/var/log/radsecproxy/radsecproxy.log \ 
      -v /path/to/radsecproxy.conf:/etc/radsecproxy.conf \
      spgreen/eduroam-radsecproxy:1.7.1-xeap2
      
A hexadecimal string indicates that the container has been created; it will be different to the one below.
```
d81ef00df0341920af99e5a8a5b7ca1cea79cd17c06cc858380e3d61d963f879
```

Check that the Docker container is running:
```
$ sudo docker ps
```
The ouput will be similar to the following if the container successfully started:
```
CONTAINER ID        IMAGE                                           COMMAND                  CREATED             STATUS              PORTS               NAMES
d81ef00df034        spgreen/eduroam-radsecproxy:1.7.1-xeap2         "/bin/sh -c '/sbin/r…"   5 minutes ago       Up 5 minutes        1812-1813/udp       nrs
```

**Viewing Logs**

Read from the log file that was mounted to the container at start-up.

```$ tail -f /path/to/log/file.log```

**The NRS is ready and now listening for RADIUS requests in the background.**
