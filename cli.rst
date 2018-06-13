====================
Command Line Options
====================

.. _cli:

AppSwitch features are provided through the ``ax`` command.

This guide goes over various options supported by ``ax`` command thereby
describing the related features.  The top level commands include:
::

     daemon		Start the AppSwitch daemon
     run		Run an application under AppSwitch
     get		Get active resources
     create		Create a resource
     delete, del	Delete a resource


Daemon
======

The ``daemon`` sub command is used to start the AppSwitch daemon.


System Call Forwarding Driver
-----------------------------
::

   --driver name	System call forwarding driver name ('user' or 'kernel')

AppSwitch can be used with either a user-space driver or a kernel-space
driver.  If the `--driver` flag is not set then the default behavior is to
use the kernel driver if the AppSwitch kernel module is loaded.  If the
module is not loaded then user-space driver is used.


Node Interface
--------------
::

   --node-interface interface		Node interface to use by daemon. Accepts IP address or interface name, eg eth0

The host interface that AppSwitch daemon should use to communicate with its peers on other nodes.
You can specify the interface by name (eg ``eth0``) or by IP address.  If unspecified, it defaults to the first interface listed on the node.


Neighbors
---------
::

      --neighbors csv		List (csv) of IP addresses of cluster neighbors

In order to configure more than one node into an AppSwitch cluster you may
provide to each node a comma separated list of IP addresses of neighbor nodes.
::

   $ ax daemon --node-interface 192.168.0.2 --neighbors 192.168.0.2,192.168.0.3


Clean Start
-----------
::

   --clean			Remove any saved state from previous sessions

When AppSwitch daemon is shutdown, the state of sockets maintained on
behalf of the applications is saved under /var/run/appswitch.  If this
flag is set, then any saved state from earlier runs of the daemon is
removed before starting the daemon. The default behavior is to restore the saved state.


Node Name
---------
::

   --node-name name		Name of the node (defaults to host name)

Name used to identify the node.  Defaults to the host name of the node
that the daemon is running on.


Profiling AppSwitch
-------------------
::

   --cpu-profile file	Write CPU profile output to file
   --mem-profile file	Write memory profile output to file

Golang profiling can be enabled using the ``--cpu-profile`` and
``--mem-profile`` options.  Each option sets the file name (relative or
absolute path) for profiling output.  Output file is a binary file that can be
viewed using standard ``go`` tools.
::

    $ ax daemon --cpu-profile /tmp/cpu.prof
    ... Ctl-C

    $ go tool pprof /tmp/cpu.prof


Domain Name System
------------------
::

   --dns		Start the built-in DNS server (default: true)

If this flag is set then AppSwitch starts a DNS server on port 53.  The
built-in DNS server resolves the names associated with the applications run
under AppSwitch.  This option is required to use the ``--name <name>``
option to the ``run`` command.


DNS Configuration Options
~~~~~~~~~~~~~~~~~~~~~~~~~
::

   --dns-domain name	DNS domain name suffix (default: "appswitch.local")
   --dns-servers csv	List (csv) of 'IP[:port]' strings of forwarding servers

Configuration options for the built-in DNS server.  You can
optionally specify a DNS suffix and a list of forwarding DNS servers.  For example
::

   $ ax daemon --dns-servers 127.0.0.1:5533,8.8.8.8


.. _tls:

Transport Layer Security
------------------------

AppSwitch can be configured to use Transport Layer Security for internal
communication among the daemons.
::

   --tls	 		Enable Transport Layer Security


TLS Configuration Options
~~~~~~~~~~~~~~~~~~~~~~~~~
::

   --tls-ca-cert file		Path to TLS CA certificate file
   --tls-cert file		Path to TLS certificate file
   --tls-key file		Path to TLS key file

Currently AppSwitch needs the absolute path to all files when configuring
TLS.  If you create a self signed certificate you can start the AppSwitch
daemon like this
::

   $ ax daemon --tls \
       --tls-ca-cert /etc/ssl/certs/cacert.pem \
       --tls-cert /etc/ssl/certs/ax.crt \
       --tls-key /etc/ssl/private/ax.key


Ports
-----
::

   --ports csv	    	List (csv) of ports, or port ranges, reserved for applications (default: "40000-60000")

AppSwitch binds application sockets to ports on the host from this port space.
::

   $ ax daemon --ports '4000,6000-8000'


.. _rest-port-label:

REST Port Number
----------------
::

   --rest-port number		REST API port number (default: 6664)

AppSwitch exposes most of its functionality through the REST API.  Most of
the the CLI commands are simply a front end to the REST API.  This option
specifies the port number used for the REST endpoint.


.. _serf-label:

Gossip Protocol
---------------

AppSwitch uses Serf as the gossip channel.  Serf can be configured with the
following options
::

   --gossip-port number		Gossip protocol port number (default: 7946)
   --gossip-auto-discover	Auto discover neighbors


Egress Gateway
--------------
::

   --egress-gateway		Configure node as egress gateway

If this flag is set, connections to external services would be proxied
through this daemon.  However, the presence of the intermediate egress
gateway would be transparent to the client running under AppSwitch.  That
is, client would directly connect to the external service and not the
egress gateway.


.. _cluster-label:

Cluster Name
------------
::

   --cluster name		Cluster name.  Required if cluster is part of a federation

Name used to identify the cluster.  All
cluster names within a federation must be unique.  Cluster name is only
needed if this node is part of a cluster that will be part of a federation
of clusters.  Otherwise the default 'appswitch' can be used.  All nodes
within the cluster should be configured with the same name.


.. _federation-label:

Federation
----------

Multiple AppSwitch clusters may be connected together to form a federation
(see :ref:`hierarchy`).  To achieve this one or more nodes in each cluster must
be configured as a federation gateway node.  Connections to services from one
cluster to another will be made through the federation gateway nodes.

A federation gateway node has two listening services.  One, referred to as
the egress federation gateway service, accepts connections from other
cluster nodes.  Data flows out of a cluster via the egress federation
gateway service.  The second, referred to as the ingress federation gateway
service accepts connections on the wide area network from other federation
gateway nodes.  Data flows into a cluster via the ingress federation
gateway service.


Federation Gateway Node Configuration Options
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
::

   --federation-gateway-ip interface		IP address or interface name for federation connectivity
   --federation-gateway-advertise-ip value	Required iff proxy node is behind NAT
   --federation-gateway-port number		TCP port number for federation gateway sessions (default: 6660)
   --federation-gateway-gossip-port number	Federation gossip protocol port number (default: 7947)
   --federation-gateway-neighbors csv		List (csv) of IP addresses of federation neighbors (other gateway nodes)


Please note also; when configuring a federation each and every node must be
configured with it's cluster name, and furthermore cluster names must be
unique within a federation (See `cluster-label`_ for details).


Run
===

The ``run`` sub command is used to run an application under AppSwitch.


IP Address
----------
::

   --ip address		IPv4 address at which services of this application would be reachable

The specified IP address is associated with the application.  When an
AppSwitch-managed client connects to the IP address, it would be
automatically directed to the services of this application.  To achieve
that, a `vservice`_ is implicitly created.  The same IP address could be
used for other applications, in which case, all those applications become
backends for the vservice.


DNS Resolvable Name
-------------------
::

   --name value		DNS resolvable name of the application

The specified name is associated with the application.  When an
AppSwitch-managed client looks up this name, it is resolved to the IP
address associated with the application by AppSwith daemon's built-in DNS
server.


Labels
------
::

   --labels csv		Labels of this application (default: "zone=default")

Allows arbitrary labels of the form ``label=value`` to be associated with
the application.  This option accepts a comma separated list of labels all
of which will be associated with the application.  Accepts arbitrary string
values for both 'label' and 'value'.  A client would be able to reach a
service only if they share at least one matching label.  For example, a
client with a label ``role=test`` cannot connect to a service with a label
``role=prod`` or one without any labels.


Exposed Ports
-------------

``--expose`` option is used to expose an internal application port on the cluster node(s) such that the service can be accessed by external non-AppSwitch clients.  There are three variations of it:

::

   --expose internal-port:host-port


The specified application port would be exposed on the specified external port only on the node where the application is running.

For example, a python web server (port 8000) can be exposed on external port 9999 as follows:

::

   $ ax run --expose '8000:9999' python -m http.server
   $ curl -I 192.168.0.2:9999
   HTTP/1.0 200 OK
   Server: SimpleHTTP/0.6 Python/3.5.2
   Date: Mon, 30 Apr 2018 05:23:33 GMT
   Content-type: text/html; charset=utf-8
   Content-Length: 2377

::

   --expose internal-port:<node-IP>:node-port

The specified application port would be exposed on the specified external port only on the specified node.  The specified node-IP must belong to one of the nodes in the AppSwitch cluster.


::

   --expose internal-port:0.0.0.0:node-port


The specified application port would be exposed on the specified external port on every node in the AppSwitch cluster. This is equivalent to the nodePort feature of Kubernetes.  A similar result can also be produced by creating an external `vservice`_.


User
----
::

   --user name		UID or user name to run the child process

When the client runs an application it is run by default as the same user
that invoked AppSwitch.  If AppSwitch is run as root (which is required to
create a new network namespace) then the application being run will be run
as root.  This is often *not* the desired behavior.  Using the ``--user``
option the name or UID of a valid user can be given to the client and the
application being run will be run as that user.
::

   $ ax run -- whoami
   root

   $ ax run --user alice whoami
   alice


Interface Name
--------------
::

   --interface name	Name of the dummy interface created within the application's network namespace

Some applications require the presence of a non-loopback network interface
in order to function.  AppSwitch places the application in a new network
namespace by default.  With this option, a dummy interface with the
specified name can be created in the new network namespace before the
application is executed.  A new network namespace has, by default, only the
loopback interface. This option requires that ``--no-new-netns`` flag is
not used.
::

   $ ax run -- ip addr show
   1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00

::

   $ ax run --interface eth0 ip addr show
   1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN group default qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
   2: eth0: <BROADCAST,NOARP,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN group default qlen 1000
    link/ether d2:a0:cb:e5:b0:33 brd ff:ff:ff:ff:ff:ff
    inet 192.168.178.2/32 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::d0a0:cbff:fee5:b033/64 scope link
       valid_lft forever preferred_lft forever


Network Namespace
-----------------
::

   --new-netns		Create a new network namespace (default: true)

Each application run by AppSwitch is run in a separate namespace.  Creation
of a new namespace is handled by AppSwitch by default.  Sometimes this
behavior is not required, for example when running within a Docker
container.  Note that creating a new network namespace also requires
privilege.  To prevent from new network namespace from being created
``--new-netns=false`` can be used.


DNS Override
------------
::

   --dns-override	Take over application's DNS requests (default: true)

This option overrides existing resolv.conf file for the application with
one that points to the built-in DNS server by mounting over it.  Host is
not affected by this.  To prevent dns override use ``--dns-override=false``.


Get
===

The ``get`` command is used to display current AppSwitch resources.

Examples:

- The IP address of the host machine is 192.168.178.2
- The daemon was started with: ``ax daemon --node-name node1``
- Two AppSwitch client instances were started
  - ``ax run -- nc -l 6000``
  - ``ax run -- iperf3 -s``



``ax get apps``
----------------

Displays information about applications currently running under AppSwitch
::

                   NAME                    APPID    NODEID   CLUSTER        APPIP     DRIVER     LABELS          ZONES
  ------------------------------------------------------------------------------------------------------------
  <ab856b81-7db0-4d88-8a1e-1bfbf0c5fe9f>  f00001bb  node1   appswitch   192.168.178.2  user    zone=default  [zone==default]
  <04a275bc-b9b4-4496-9c9d-a838daecdffb>  f000028e  node1   appswitch   192.168.178.2  user    zone=default  [zone==default]


``ax get servers``
-------------------

Shows information about currently running services.
::

	  NODEID   CLUSTER     APPID    PROTO     SERVICEADDR           IPV4ADDR
          --------------------------------------------------------------------------
          node1   appswitch   f00001bb  tcp    192.168.178.2:6000  10.0.23.11:40000
          node1   appswitch   f000028e  tcp    192.168.178.2:5201  10.0.23.11:40001

SERVICEADDR above represents the virtual IP address where the service is
available to AppSwitch-managed clients and the IPV4ADDR represents the host
IP and port where the service is actually bound.


``ax get sockets``
-------------------

Displays socket information for currently running applications
::

                      ID                   NODEID    APPID   INODE  PROTO  FLAGS     BINDIP      BACKLOG
     ----------------------------------------------------------------------------------------------------
     4daa64c4-2091-46e0-8a67-5428fae9775d  node1   f00001bb  829    tcp    0      0.0.0.0:6000   1
     f6dd4f27-bb4e-4c4a-a076-dc07c29af7be  node1   f000028e  833    tcp    0      0.0.0.0:5201   5


``ax get proxies``
-------------------

Displays current proxies.  Example listing is off a node configured to be
a federation gateway node (see :ref:`federation-label` for details).
::

	  ID  PROTO       LISTENER         DIALERS    
	--------------------------------------------
	  1   tcp    10.0.0.10:6660      [0.0.0.0:0]  
	  2   tcp    192.168.0.10:36869  [0.0.0.0:0] 


Create
======

Create command is used to create a resource.  Resource is a general
construct that represents a particular AppSwitch feature.

Currently supported resources are:

.. _vservice-label:

::

	vservice		Create a virtual service


vservice
--------

A virtual service (vservice) is a virtual-IP:virtual-port combination
that acts as a load balancing front end to a set of backend services.
Backend services consist of services listed in the service table or
external services specified as IP:port pairs.  The vIP, vPort and the
IP:Ports of the backend services are specified by the user.

The following options are provided by vservice command.
::

   --ip value		IPv4 address for the virtual service
   --external		Make this vservice external

A virtual service can be marked external.  In that case, in addition
to creating the vservice with the specified vIP and vPort, the virtual
service represented by the load balanced backend services is exposed on
the vPort on all nodes of the cluster.
::

   --lbtype value	Load Balancer Type <possible values: Random, RoundRobin> (default: "Random")
   --backends value	Comma separated list of IPv4 IPs
   --ports value	Comma separated list of <virtual port:application port>. The service will be made available on all cluster nodes on virtual port
   --source-ip		Host IP address to use when making the outbound connection


Delete
======


