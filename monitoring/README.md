#Monitoring

## Introduction

We want to do some basic monitoring on both the host OS and the containers. Nagios is the obvious choice because it's probably
the most popular monitoring software and supports remote monitoring. We'll be using Nagios Version 3 as it's the default supported version in Ubuntu 12.04.

We don't want to install the full Nagios3 package on the host OS or containers as it has many depedencies we'd rather not install, like Apache. Instead, we're going to the use the `nagios-nrpe-server` package to run checks on the remote servers, and install the full Nagios3 web package onto a seperate machine. 

We'll be using the backup machine for installing the main nagios web package and runnng checks on the remote servers.

## Installing Nagios on the monitoring/backup server

Before we setup monitoring on the host OS and containers, we need to install nagios on our monitoring/backup server:

```
sudo apt-get update
sudo apt-get install nagios3 nagios-nrpe-plugin
```

Navigate to monitoringserver/nagios3 with your browser.

## Ignore backup mounts on the monitoring server

We want to ignore SSHFS mounted drives on the monitoring server.

Open up `/etc/nagios3/conf.d/localhost_nagios2.cfg`

Add:

```
define command{
        command_name    check_all_disks_ignore_ssfhs_mounts
        command_line    /usr/lib/nagios/plugins/check_disk -w '$ARG1$' -c '$ARG2$' -e -A -i "/backup/proxima"
}
```

Adjust the disk space service to use that command, for example:

```
define service{
        use                             generic-service         ; Name of service template to use
        host_name                       localhost
        service_description             Disk Space
        check_command                   check_all_disks_ignore_ssfhs_mounts!20%!10%
        }
```

## Installing the nagios server within a container or the host OS

```
sudo apt-get update
sudo apt-get install nagios-nrpe-server
```

Check that the plugins were installed:

```
ls -l /usr/lib/nagios/plugins/
```

### Adjusting the config

```
vi /etc/nagios/nrpe_local.cfg
```

Add the following, but change X.X.X.X to the either the ip address of the monitoring server. or the address of the network bridge, if installing in a container (eg 10.0.3.1). (I needed to add this when port-forwarding to the nagios nrpe service within containers.)
dont_blame_nrpe is set to 1 allow (n) arguments to be passed to the commands. (I am yet to figure out a better way to do this. Passing arguments feature is required for "check_http_domain" command,)

```
######################################
# Do any local nrpe configuration here
######################################

allowed_hosts=127.0.0.1,X.X.X.X
dont_blame_nrpe=1

command[check_users]=/usr/lib/nagios/plugins/check_users -w 5 -c 10
command[check_load]=/usr/lib/nagios/plugins/check_load -w 15,10,5 -c 30,25,20
command[check_rootfs_disk]=/usr/lib/nagios/plugins/check_disk -w 20 -c 10 /
command[check_zombie_procs]=/usr/lib/nagios/plugins/check_procs -w 5 -c 10 -s Z
command[check_total_procs]=/usr/lib/nagios/plugins/check_procs -w 150 -c 200
command[check_swap]=/usr/lib/nagios/plugins/check_swap -w 20 -c 10
command[check_apt]=/usr/lib/nagios/plugins/check_apt
command[check_smtp]=/usr/lib/nagios/plugins/check_smtp -H localhost
command[check_http]=/usr/lib/nagios/plugins/check_http -H localhost
command[check_http_domain]=/usr/lib/nagios/plugins/check_http -H $ARG1$ -u /

```

Restart the nrpe service:

```
service nagios-nrpe-server restart
```

## Forwarding ports for nagios installed in containers

If you installed the nrpe-server on a container, then we need to port-forward an 'outside' port on the host OS to an 'internal' container port. I leave the default port setting in `/etc/nagios/nrpe.cfg', which is 5666. I then port-forward ports starting from 5667 to port 5666 on the various containers.

Enable ip forwarding:

```
echo 1 > /proc/sys/net/ipv4/ip_forward

```

In /etc/sysctl.conf:

```
net.ipv4.ip_forward=1
```

Add a port forward rule:

```
iptables -t nat -A PREROUTING -p tcp -d <EXTERNAL_HOST_IP> -j DNAT --dport 5667 --to-destination <CONTAINER_IP>:5666
```

Save the rules:

```
iptables-save > /etc/iptables.conf
```

View the rules:

```
iptables -t nat -L
```

Check the port is open:

```
nmap localhost -p 5666
```

Now on your backup/monitoring server, check that we can connect to the nagios nrpe server within the container:

```
nmap X.X.X.X -p 5667
/usr/lib/nagios/plugins/check_nrpe -H X.X.X.X -p 5667
```

Adding a new host to the nagios monitoring server:

```
cd /etc/nagios3/conf.d
cp localhost_nagios2.cfg container.yourhost.com_nagios2.cfg
vi container.yourhost.com
```

Add the following: (change X.X.X.X to your host machine ip address, change <port> to the port where nagios-npre-server is listening.)

```
# A simple configuration file for monitoring the local host
# This can serve as an example for configuring other servers;
# Custom services specific to this host are added here, but services
# defined in nagios2-common_services.cfg may also apply.
# 

define host{
        use                     generic-host            ; Name of host template to use
        host_name              	container.yourhost.com
        alias                   container.yourhost.com
        address                 X.X.X.X -p <port>
        #If container:
        #parents                 example.com
        }

# Define a service to check the disk space of the root partition
# on the local machine.  Warning if < 20% free, critical if
# < 10% free space on partition.

define service{
        use                             generic-service         ; Name of service template to use
        host_name                       container.yourhost.com
        service_description             RootFS Disk Space
        check_command                   check_nrpe_1arg!check_rootfs_disk
        }



# Define a service to check the number of currently logged in
# users on the local machine.  Warning if > 20 users, critical
# if > 50 users.

define service{
        use                             generic-service         ; Name of service template to use
        host_name                       container.yourhost.com
        service_description             Current Users
        check_command                   check_nrpe_1arg!check_users
        }


# Define a service to check the number of currently running procs
# on the local machine.  Warning if > 250 processes, critical if
# > 400 processes.

define service{
        use                             generic-service         ; Name of service template to use
        host_name                       container.yourhost.com
        service_description             Total Processes
	check_command                   check_nrpe_1arg!check_total_procs
        }



# Define a service to check the load on the local machine. 

define service{
        use                             generic-service         ; Name of service template to use
        host_name                       container.yourhost.com
        service_description             Current Load
	check_command                   check_nrpe_1arg!check_load
        }
        
# Define a service to check HTTP status of domains
define service{
        use             generic-service
        host_name       milos.proxima.cc
        service_description your_domain.com HTTP
        check_command check_nrpe!check_http_domain!"your_domain.com"
}

```

You can group containers into a hostgroup.

In /etc/nagios3/conf.d/hostgroups_nagios2.cfg, add a block:

```
define hostgroup {
        hostgroup_name containers
        alias LXC containers
        members         container.example.com,container2.example.com
}
```


Restart nagios:

```
service nagios3 restart
```
