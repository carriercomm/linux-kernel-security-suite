# Netlog

Netlog is a Loadable Kernel Module that logs information for every connection on the hosted machine.
By utilizing the kprobes API, it probes the connect and accept system calls ,for TCP connections,
as well as the bind system call, for UDP connections, and the socket destruction. 
While probing, it logs the process name and process id, the user id that owns this process, the protocol
(TCP/UDP), the local IP address and port as well as the remote IP address and port.

For UDP, it tracks only the bind system call, because UDP is a connectionless protocol,
so the other approach was to track the packet communication, something that would
add a lot of overhead. In order to compile the code that tracks UDP, you 
have to change the symbolic constant (PROBE_UDP) that it's defined in the netlog header (netlog.h) from
zero (0) to a non-zero value (i.e. 1) and recompile netlog.

netlog also tracks the destruction of the sockets. This is done by probing the 
inet_shutdown kernel call, which is called after the close system call is called.
In order to compile the code that tracks the connection close, you need to change the value of the
symbolic constant (PROBE_CONNECTION_CLOSE) that is defined in the netlog header file (netlog.h) from 
zero (0) to a non-zero value (i.e. 1) and recompile netlog.

# Kernels tested against

2.6.18 up to 3.4.9

# Change Log

1.30
	*On the fly configuration of the whitelist

1.12
	*Optimizations
	*Dynamic Whitelisting
1.15
	*Whitelisting of ip addresses and ports
	*Absolute path mode
1.16
	*Basic checks on given ip addresses and ports
	
# Dynamic Whitelisting

netlog offers whitelisting of connections at insertion time. This is done by passing parameters to the module.
The parameter is named connections_to_whitelist and it's an array of strings with the following format:

"/absolute/execution/path|i<ip address>|p<port number>"

i.e.:
	"/usr/sbin/sshd|i<127.128.0.0>|p<22>"
	"/usr/sbin/sshd|i<127.128.0.0>"
	"/usr/sbin/sshd|p<22>"
	"/usr/sbin/sshd"

The '<' and '>' can be ignored, they are supported just for visual reasons.

Example of whitelisting multiple connections:

insmod netlog.ko connections_to_whitelist="/usr/bin/skype","/usr/bin/ssh|i<127.0.0.1>","/usr/bin/sshd|i<127.0.0.1>|p<22>"

Note that you are not able to whitelist an ip address or port without specifing an absolute 
execution path of a binary. Also, you can whitelist up to 150 connections. For more, change
the symbolic constant MAX_WHITELIST_SIZE in whitelist.h and recompile netlog.
Do not leave spaces between the comma separators and the strings because the parameter array will not be initialized.

If you believe that you will not need whitelisting at all, we recommend you to compile netlog with disabled the whitelisting
in order to boost even more its perfomance. In order to not compile the code of whitelisting, change the static constant
WHITELISTING to 0 in netlog.h file and recompile netlog.

# On the fly configuration of the whitelist

The process of whitelisting (described in "Dynamic Whitelisting" section) can be done also while netlog is planted.
If you want to change the whitelist when netlog is planted, there is no need to remove the module and insert 
it again with the updated whitelist information passed as insertion parameters. This can be done by editing the
/proc/config-netlog proc file. This file contains per line the currently whitelisted connections. 

i.e. if you want to whitelist the next connections, after insertion of netlog:

"/usr/bin/skype", "/usr/bin/ssh|i<127.0.0.1>" and "/usr/sbin/sshd|i<127.0.0.1>|p<22>"

You should execute in a bash shell:

echo "/usr/bin/skype,/usr/bin/ssh|i<127.0.0.1>,/usr/sbin/sshd|i<127.0.0.1>|p<22>" > /proc/netlog_config

* **************************************WARNING**************************************** *
*											*
* Do not edit the proc_config entry with a text editor or try to replace it etc.	*
* Just echo the connection strings that describes the connections you want to whitelist,*
* separated by the comma character and NOTHING MORE (not even a space).			*
* Also do not try to append the proc file.						*
*											*
* For the format of the connection string read the "Dynamic Whitelisting" section above.*
* ************************************************************************************* *

netlog will log the changes of the whitelist:

Sep 19 09:57:26 panos-PC kernel: [ 1841.262471] netlog-config:	[+] Cleared whitelist
Sep 19 09:57:26 panos-PC kernel: [ 1841.262479] netlog-config:	[+] Whitelisted /usr/sbin/skype
Sep 19 09:57:26 panos-PC kernel: [ 1841.262483] netlog-config:	[+] Whitelisted /usr/bin/ssh|i<127.0.0.1>
Sep 19 09:57:26 panos-PC kernel: [ 1841.262486] netlog-config:	[+] Whitelisted /usr/sbin/sshd|i<127.0.0.1>

# Absolute Path Mode

The absolute path mode is activated by passing a non zero value to the absolute_path_mode parameter.

i.e.:
	insmod netlog.ko absolute_path_mode=1

Absolute path mode will log the absolute execution path instead of the process name.
This will be helpfull in case that you want to see the absolute execution path of a connection that
you want to whitelist.

# Format of log messages

TCP connect:
Dec 19 14:03:17 panos-PC kernel: netlog: http[3206] TCP 137.138.191.167:52507 -> 130.59.10.36:80 (uid=0)

TCP accept:
Dec 19 14:18:37 panos-PC kernel: netlog: sshd[827] TCP 137.138.191.167:22 <- 137.138.32.18:49904 (uid=0)

UDP binds:
Dec 19 14:22:50 panos-PC kernel: netlog: skype[4261] UDP bind 127.0.0.1:0 (uid=1000)
Dec 19 14:22:50 panos-PC kernel: netlog: skype[4261] UDP bind (any IP address):2752 (uid=1000)

Connection close:
Dec 19 14:19:07 panos-PC kernel: netlog: sshd[3755] TCP 137.138.191.167:22 <-> 137.138.32.18:49904 (uid=0)

# How to use

In the src directory run:

	make

Then run:

	insmod netlog.ko

in order to insert the module in the kernel.
And in order to remove it, run:

	rmmod netlog

# License

Copyright 2011 CERN. This software is distributed under the terms of the GNU General Public
Licence version 3 (GPL Version 3), copied verbatim in the file COPYING. In applying this licence,
CERN does not waive the privileges and immunities granted to it by virtue of its status as an 
Intergovernmental Organization or submit itself to any jurisdiction.

# References

[Kernel instrumentation using kprobes](http://www.phrack.org/issues.html?issue=67&id=6)
