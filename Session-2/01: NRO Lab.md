# XeAP 2 Session 2: NRO Lab<br>Setting up an eduroam National RADIUS Server


The XeAP 2 project will be using radsecproxy as the National and Top RADIUS Servers within the XeAP2 eduroam environment. There are other alternatives - such as FreeRADIUS - that are able to perform as an NRS but radsecproxy is lightweight and only requires the user to worry about a single configuration file unlike FreeRADIUS.

  
As the XeAP 2 Virutal Machines are running Ubuntu 18.04 LTS, we will install radsecproxy v1.7.1 from source as the apt package manager only provides version 1.6.9.


## radsecproxy Installation

1.  Update and refresh the Ubuntu package repositories to ensure we receive the most up-to-date packages:

		$ sudo apt update   

2. Upgrade currently installed packages via apt package manager:

		$ sudo apt upgrade

3. Install packages needed to compile radsecproxy from source:

		$ sudo apt install build-essential libssl-dev make nettle-dev curl
		
4. Change directory to ```/usr/local/src/``` and download the radsecproxy v1.7.1 release:
		
		$ cd /usr/local/src/
		$ sudo curl -Lo radsecproxy-1.7.1.tar.gz \
		      https://github.com/radsecproxy/radsecproxy/releases/download/1.7.1/radsecproxy-1.7.1.tar.gz

5. Extract the radsecproxy package, remove the package and enter into the radsecproxy source directory:
		
		$ sudo tar xpvf radsecproxy-1.7.1.tar.gz
		$ sudo rm radsecproxy-1.7.1.tar.gz
		$ cd radsecproxy-1.7.1

6. Download and apply the radsecproxy patch needed to successfully compile the package on Ubuntu 18.04 LTS:

		$ sudo curl -fsL -o tests/t_fticks.patch \
		    "https://raw.githubusercontent.com/spgreen/eduroam-radsecproxy-docker/master/1.7.1-xeap2/patch/tests/t_fticks.patch"
		$ sudo patch tests/t_fticks.c tests/t_fticks.patch

7. Download the line log patch which adds the Operator Name and Chargeable User Identity (CUI) attributes to the radsecproxy logs. Shout-out to Vlad Mencl, REANNZ, for creating the patch.

		$ sudo curl -fsL -o radsecproxy-log-opname-cui.diff \
		    "https://raw.githubusercontent.com/spgreen/eduroam-radsecproxy-docker/master/1.7.1-xeap2/patch/radsecproxy-log-opname-cui.diff"
		$ sudo patch -p1 < radsecproxy-log-opname-cui.diff

8. Configure, compile, check and install radsecproxy -v:

		$ sudo ./configure
		$ sudo make
		$ sudo make check
		$ sudo make install

9. radsecproxy is now installed. The executable can be found using the ```which``` command:
	
		$ which radsecproxy
		
	Output:
	
		/usr/local/sbin/radsecproxy
		
10. Check the radsecproxy version once installation has been completed:

		$ radsecproxy -v 

	Output you should receive:
	```
	radsecproxy revision 1.7.1
	This binary was built with support for the following transports:
	  UDP
	  TCP
	  TLS
	  DTLS
	```
	
11. Create an empty configuration file at the default radsecproxy configuration file location:

		$ sudo touch /usr/local/etc/radsecproxy.conf
		
12. Create the log directory and the log file where radsecproxy will store its logs:

		$ sudo mkdir /var/log/radsecproxy
		$ sudo touch /var/log/radsecproxy/radsecproxy.log

13. Open ```/usr/local/etc/radsecproxy.conf``` in your favourite text editor (vim, nano, emacs, etc):

		$ sudo vim /usr/local/etc/radsecproxy.conf
		
    **You are now ready to create the configuration for your new National RADIUS Server!**


## radsecproxy Configuration


1. Within the editor, add port and logging configuration:

	- **Listening interface and port number:**

	```
	# Listen on UDP port 1812 for all interfaces
	ListenUDP *:1812
	```

	- **General Logging:**

	```
	# Creates linelogs
	LogLevel 3
	
	# Write logs to /var/log/radsecproxy/radsecproxy.log file
	LogDestination file:///var/log/radsecproxy/radsecproxy.log
	```
	
	- **Loop prevention:**

		Semi automatic prevention loop for RADIUS connections. Defining a client and server block with the same descriptive name will prevent radsecproxy from proxying request to the same server. This is to ensure that only roaming user requests are sent to the proxy. <br>
	 	
		**Important Note:** You may wish to set this as ```	LoopPrevention Off``` for the XeAP2 workshop as loopbacks will be used for debugging purposes. 

	```
	LoopPrevention On 
	```
	
	- **Remove VLAN attributes:**

		Removes VLAN attributes that can be accidentally sent to the FLR even though it should be filtered out by the Service Provider (SP) before reaching the NRS. Identity Providers (IdPs) should not send this out at all. If the block was not in use and there were existing VLAN attributes, then the user would experience degraded or non-existent service.
	
    ```
    rewrite defaultclient {
        # IETF 64 (Tunnel Type)
        removeAttribute 64
        # IETF 65 (Tunnel Medium Type)
        removeAttribute 65
        # IETF 81 (Tunnel Private Group ID)
        removeAttribute 81
     }
     ```

2. Add Institutional RADIUS Servers (IRS) using Client, Server and Realm blocks. An IRS can act as an IdP, SP, or IdP & SP:
	
	- **Client Blocks**
	
		Client blocks are needed for eduroam Service Providers. You must communicate with the SP in regards to a secret so that the SP RADIUS server can communicate with the National RADIUS Server.

    ```
    client IHL-1 {
        # IP address or Fully Qualified Domain Name (FQDN) of the SP - example used
        host 203.0.113.1
        # RADIUS communication protocol used
        type UDP
        # Secret key negotiated between NRO and Institution
        secret changeme
    }
    ```

	- **Server Blocks**
	
		Server blocks are used for forwarding Access-Requests to an eduroam Identity Provider. Since the majority of IRS' will be both an IdP and a SP together, you will need to create a client and server block for each unique IRS.
    
    ```
    server IHL-1 {
        host 203.0.113.1
        type UDP
        secret changeme
        # Periodically sends status checks to the IRS server
        statusserver on
   }
   ```

	- **Realm Blocks**
	
		Realm blocks indicate which IRS a roaming user's authentication request needs to be sent to. radsecproxy analyses the roaming user's realm (e.g. ```@ihl1-domain.edu.sg```)  from an authentication request, matches it with a Realm block and then sends it to the IRS belonging to said block. 
		
		Realm blocks use regular expressions to match patterns found within a realm. 
		For the realm block example below, it matches  ```@ihl1-domain.edu.sg``` and all sub-domains of ```ihl1-domain.edu.sg``` by using the following regular expression before the base domain: ```/(@|\.)```.
	
	```
	realm /(@|\.)ihl1-domain.edu.sg {
            server IHL-1
	}
	```
	**Block Requirements**
		
	- **IRS as an Identity and Service Provider** - Client, Server and Realm blocks are required
	- **IRS as an Identity Provider** - Server and Realm blocks are required
	- **IRS as a Service Provider** - Only Client blocks are required


3. Add the blacklist filters:

	- Uses regular expressions to check realm and since no server is specified, it will send back an Access-Reject packet with a replymessage to notify the Service Provider on why the Access-Request was rejected. This helps prevent bogus Access-Requests from being forwarded to the eduroam TLR.
 
	```
	realm /\s/ {
      replymessage "Misconfigured client: no route to white-space realm! Rejected by NRS."
	}

	realm /^[^@]+$/ {
      replymessage "Misconfigured client: no AT symbol in realm! Rejected by NRS."
	}

	realm /myabc\.com$ {
      replymessage "Misconfigured supplicant: default realm of Intel PRO/Wireless supplicant!"
	}

	realm /^$/ {
      replymessage "Misconfigured client: empty realm! Rejected by <TLD>."
      accountingresponse on
	}
	
	realm /(@|\.)outlook.com {
      replymessage "Misconfigured client: invalid eduroam realm."
      accountingresponse on
	}

	realm /(@|\.)live.com {
      replymessage "Misconfigured client:  invalid eduroam realm."
      accountingresponse on
	}

	realm /(@|\.)gmail.com {
      replymessage "Misconfigured client: invalid eduroam realm."
      accountingresponse on
	}

	realm /(@|\.)yahoo.c(n|om) {
      replymessage "Misconfigured client: invalid eduroam realm."
      accountingresponse on
	}
	```
4. Add the Top Level RADIUS (TLR) Client and Server blocks; Realm block will be added at the end.
	
	```
	# eduroam Top Level RADIUS blocks 
	
	client eduroam_TLR_1 {
	    type UDP
	    # Example IP address - Needs to be changed
	    host 198.51.100.1
	    secret __eduroam_secret-CHANGE-ME__
	}
	server eduroam_TLR_1 {
	    type UDP
	    # Example IP addres - Needs to be changed
	    host 198.51.100.1	
	    secret __eduroam_secret-CHANGE-ME__	
	    statusserver on
	}

	client eduroam_TLR_2 {
	    type UDP
	    # Example IP address - Needs to be changed	    
	    host 198.51.100.2	
	    secret __eduroam_secret-CHANGE-ME__
	} 
	server eduroam_TLR_2 {
	    type UDP
	    # Example IP address - Needs to be changed
	    host 198.51.100.2
	    secret __eduroam_secret-CHANGE-ME__
	    statusserver on
	}
	```
5. Add the eduroam Top Level RADIUS (TLR) Realm Block

	- The eduroam TLR Realm block forwards all other authentication requests to the TLR to perform additional routing if it is unable to find an appropriate IRS to authenticate the roaming user.

	```
	# DEFAULT forwarding: to the Top-Level Servers
	
	realm * {
	    server eduroam_TLR_1
	    server eduroam_TLR_2
	}
	```

**Example /etc/radsecproxy.conf**

```
#######################################################
#          GENERAL AND LOGGING CONFIGURATION          #
#######################################################
ListenUDP *:1812
LogLevel 3
LogDestination file:///var/log/radsecproxy/radsecproxy.log

LoopPrevention On 

# Remove VLAN attributes
rewrite defaultclient {
    removeAttribute    64
    removeAttribute    65
    removeAttribute    81
}


#######################################################
#     eduroam IRS CLIENT, SERVER and REALM BLOCKS     #
#######################################################

# IHL-1 IdP and SP Block
client IHL-1 {
    host              203.0.113.1
    type              UDP
    secret            changeme
}	
server IHL-1 {
    host              203.0.113.1
    type              UDP
    secret            changeme
    statusserver      on
}
# IHL-1 Realm
realm /(@|\.)ihl1-domain.edu.sg {
	server	IHL-1
}


#######################################################
#              BLACKLIST REALM FILTERS                #
#######################################################
realm /\s/ {
    replymessage "Misconfigured client: no route to white-space realm! Rejected by NRS."
}

realm /^[^@]+$/ {
    replymessage "Misconfigured client: no AT symbol in realm! Rejected by NRS."
}

realm /myabc\.com$ {
    replymessage "Misconfigured supplicant: default realm of Intel PRO/Wireless supplicant!"
}

realm /^$/ {
    replymessage "Misconfigured client: empty realm! Rejected by <TLD>."
    accountingresponse on
}

realm /(@|\.)outlook.com {
    replymessage "Misconfigured client: invalid eduroam realm."
    accountingresponse on
}

realm /(@|\.)live.com {
    replymessage "Misconfigured client:  invalid eduroam realm."
    accountingresponse on
}

realm /(@|\.)gmail.com {
    replymessage "Misconfigured client: invalid eduroam realm."
    accountingresponse on
}

realm /(@|\.)yahoo.c(n|om) {
    replymessage "Misconfigured client: invalid eduroam realm."
    accountingresponse on
}


#######################################################
#        eduroam TLR CLIENT and SERVER BLOCKS         #
####################################################### 

client eduroam_TLR_1 {
        type UDP
        host 198.51.100.1
        secret __eduroam_secret__
} 
server eduroam_TLR_1 {
        type UDP
        host 198.51.100.1
        secret __eduroam_secret__
        statusserver on
}

client eduroam_TLR_2 {
        type UDP
        host 198.51.100.2
        secret __eduroam_secret__
} 
server eduroam_TLR_2 {
        type UDP
        host 198.51.100.2
        secret __eduroam_secret__
        statusserver on
}

#######################################################
#                  TLR REALM BLOCK                    #
#######################################################

# DEFAULT forwarding: to the Top-Level Servers
realm * {
    server eduroam_TLR_1
    server eduroam_TLR_2
}
```

## Starting the National RADIUS Server 

There are two settings that are used to run the radsecproxy service. The first is in 'foreground' mode which we will use for debugging purposes. It will display logs to stdout as shown below.

1. Verify the configuration file is syntactically fine by running radsecroxy in pretend mode. Output will only appear if there is an issue with the configuration file.

	```
	$ sudo radsecproxy -p	
	```
	
	
2. Run radsecproxy in foreground mode. Logs will be sent to stdout instead of the log file stated in ```/etc/radsecproxy.conf```. 

	```
	$ radsecproxy -f
	```
	```
	Jul 17 00:00:04 2018: createlistener: listening for udp on *:1812

	```


3. Stop radsecproxy by using ```Ctrl + c``` and start radsecproxy in the background.

	```
	$ sudo radsecproxy
	```

4. Check the log file to ensure radsecproxy logs are being received.

	```
	$ cat /var/log/radsecproxy.log
	```
	```
	Jul 11 00:00:04 2018: createlistener: listening for udp on *:1812

	```
  **Your NRS is now ready for production use.**

## Shutting Down radsecproxy

	$ sudo pkill radsecproxy


## Starting radsecproxy Automatically After Server Start-Up

1. If `/etc/rc.local` is not present, creat the `/etc/rc.local` file and make it excutable:

		$ sudo touch /etc/rc.local
		$ sudo chmod +x /etc/rc.local
	
2. Open `/etc/rc.local` with your favourite text editor (vim, nano, emacs, etc):

		$ sudo vim /etc/rc.local

3. Add contents to `/etc/rc.local` 
	
	- If `/etc/rc.local` is a new file, add the following:

			#!/bin/sh -e
			#
			# rc.local
			#
			# This script is executed at the end of each multiuser runlevel.
			# Make sure that the script will "exit 0" on success or any other
			# value on error.
			#
			# In order to enable or disable this script just change the execution
			# bits.
			#
			# By default this script does nothing.

			radsecproxy
			exit 0
		
	- Otherwise add the radsecproxy start command between `#!/bin/sh -e` and `exit 0` lines as shown above.
	
**radsecproxy will now start once the server boots up.**


## Debugging Tool Overview

### tcpdump
A network packet analyser tool that can be used to view incoming and outgoing packets on a network interface in real-time. We will use tcpdump to check on RADIUS communication happening between eduroam servers.

**Example Command**:
```
$ sudo tcpdump -i ens160 port 1812 -T radius
```
```
...
13:07:48.652516 IP sg-nrad1.tein.aarnet.edu.au.radius > sg-rad1.tein.aarnet.edu.au.36217: RADIUS, Access-Challenge (11), id: 0x07 length: 140
13:07:48.656860 IP sg-rad1.tein.aarnet.edu.au.36217 > sg-nrad1.tein.aarnet.edu.au.radius: RADIUS, Access-Request (1), id: 0x08 length: 189
13:07:48.656964 IP sg-nrad1.tein.aarnet.edu.au.47767 > au-nrad1.tiein.aarnet.edu.au.radius: RADIUS, Access-Request (1), id: 0x49 length: 189
13:07:48.658624 IP au-nrad1.tiein.aarnet.edu.au.radius > sg-nrad1.tein.aarnet.edu.au.47767: RADIUS, Access-Challenge (11), id: 0x49 length: 104
...
```
**Flag and Input Explanation**
- `-i ens160`: Listen on network interface ens160. The interface name may or may not be different on your host machine.
- `port 1812`: Listen on port 1812.
- `-T radius`: Listen for only RADIUS packets.
Check the tcpdump manual page for additional options. 


### ss
ss or Socket Statistics is a tool that displays current socket information of the host machine. It allows us to check whether the radsecproxy process is listening to UDP port 1812 by using the following command:

```	
$ sudo ss -ulpn sport eq 1812
```

```
State   Recv-Q   Send-Q  Local Address:Port   Peer Address:Port
UNCONN  43520    0             0.0.0.0:1812            0.0.0.0:*   users:(("radsecproxy",pid=2668,fd=5))
UNCONN  0        0                [::]:1812               [::]:*   users:(("radsecproxy",pid=2668,fd=6))

```
The output shows that radsecproxy is indeed listening on UDP Port 1812 on both IPv4 and IPv6 on all interfaces indicated by `0.0.0.0`.

**Flag and Input Explanation**
- `-u`: Display UDP sockets
- `-l`: Display only listening sockets
- `-p`: Show process using socket
- `-n`: Displays numeric port number. If this was not used, you would see `radius` instead of `1812`
- `sport eq 1812`: Filters for source port equal to 1812

---
## Conclusion

The NRS is now setup and ready to accept RADIUS requests from trusted Institutional RADIUS Servers and the eduroam Top Level RADIUS. When new Institutions of Higher Learning join your federation, be sure to add the appropriate Client, Server and/or Realm blocks depending on what type of service their Institutional RADIUS Server provides. Always ensure that the Top Level RADIUS (TLR) Realm block is at the bottom of the configuration file. This will ensure that radsecproxy forwards only valid user requests to the eduroam TLR.


### Extra Configuration Information
For more in-depth explanations and configuration options, please visit the radsecproxy manual page located at [https://radsecproxy.github.io/radsecproxy.conf.html](https://radsecproxy.github.io/radsecproxy.conf.html "Offical radsecproxy.conf manual page").

