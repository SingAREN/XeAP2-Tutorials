# National RADIUS Server (NRS) Docker Container

The NRS Docker container uses radsecproxy on a base Ubuntu 18.04 LTS image.

## Using the NRS Docker Image

Create or use existing a radsecproxy configuration file in your desired directory; it will be mounted into the container on start-up.

Create a log file if you want logs to be saved on the Docker host machine: E.g. ```# touch /var/log/radsecproxy.log```

### Foreground Start
Start the container in the foreground using the following command. Remember to change the file paths for the radsecproxy log and configuration file.

    $ sudo docker run -it --name nrs \
      -p 1812:1812/udp \
      -p 1813:1813/udp \
      -v /etc/localtime:/etc/localtime:ro \
      -v /path/to/log/file:/var/log/radsecproxy/radsecproxy.log \ 
      -v /path/to/radsecproxy.conf:/etc/radsecproxy.conf \
      spgreen/eduroam-radsecproxy:1.7.1
      
If the configuration is correct, you will see a similar output:

    Jul 23 07:07:46 2018: createlistener: listening for udp on *:1812
    
The radsecproxy NRS Docker container will not start if there is an issue with the radsecproxy configuration file.

**The NRS is ready and now listening for RADIUS requests**

### Background Start

Once you know that the NRS Docker works, you would want to the container in the backgroud. This can be done by using the ```-d``` flag within the ```docker run``` command.

Stop and remove the current NRS Docker container. Since we used a name to initialise the container, we can stop and remove it using that same name. 

    $ sudo docker stop nrs; sudo docker rm nrs

Restart the container but this time in background mode

```
$ sudo docker run -dit --name nrs \
  -p 1812:1812/udp \
  -p 1813:1813/udp \
  -v /etc/localtime:/etc/localtime:ro \
  -v /path/to/log/file:/var/log/radsecproxy/radsecproxy.log \ 
  -v /path/to/radsecproxy.conf:/etc/radsecproxy.conf \
  spgreen/eduroam-radsecproxy:1.7.1
```
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
d81ef00df034        spgreen/eduroam-radsecproxy:1.7.1               "/bin/sh -c '/sbin/r…"   5 minutes ago       Up 5 minutes        1812-1813/udp       nrs
```

To view logs radsecproxy logs, you can run the ```docker logs``` command or read directly from the log file if you mounted a file during container start-up.

- ```$ sudo docker logs nrs```
- ```$ tail -f /path/to/log/file/used/for/mounting```

**The NRS is ready and now listening for RADIUS requests in the background.**
