========
Examples
========

Here we describe some example usage scenarios for AppSwitch.  There is a repository_ on GitHub, please clone it to follow along with these examples.

.. _repository: https://github.com/appswitch/examples/

::

   $ git clone https://github.com/appswitch/examples/
   $ cd examples


All examples require use of Vagrant to bring up the virtual machines used to form an AppSwitch cluster.  Instructions on installing Vagrant can be found on the HashiCorp_ website

.. _HashiCorp: https://www.vagrantup.com/docs/installation/


We will be progressively working up to an advanced AppSwitch configuration using all four nodes.  The node naming convention is based on the requirements of those use cases.  To start with the naming convention is not important but for the interested, the nodes are named ``hostXY`` where ``X`` is the cluster number and ``Y`` is the node number.

Bring up the first node and ssh into it:
::

   $ vagrant up host00
   $ vagrant ssh host00


The AppSwitch binary (``ax``) is distributed as an image via Docker Hub.  The latest image is already pulled as part of provisioning the Vagrant VM.


Single Node Cluster - Basic AppSwitch Usage
===========================================

First step is to bring up AppSwitch daemon with the docker-compose file.  It includes basic configuration:
::

   $ cd appswitch
   $ docker-compose -f docker-compose-host00.yaml up -d appswitch


Let's start a simple Python based HTTP server using the supplied docker-compose file.
::

   version: '2.3'

   services:
     http:
       image: python:alpine
       entrypoint: /usr/bin/ax run --name test
       command: python -m http.server
       networks: []
       volumes:
         - /usr/bin:/usr/bin
         - /var/run/appswitch:/var/run/appswitch

::

   $ docker-compose -f docker-compose-httpserver.yaml up -d httpserver


That brings up the server on the specified ip (``1.1.1.1``).  Note that the container in which the server runs has no networks.  All network connectivity is handled by AppSwitch.  Also note that it is an unprivileged container.

Let's check it with ``curl``.


::

   $ sudo ax run -- curl -I 1.1.1.1:8000
   HTTP/1.0 200 OK
   Server: SimpleHTTP/0.6 Python/3.6.5
   Date: Wed, 06 Jun 2018 01:37:55 GMT
   Content-type: text/html; charset=utf-8
   Content-Length: 882


The above command is run as root because ax creates a new network namespace to isolate the application.  ``sudo`` won't be needed with ``--new-netns=false`` option:
::

   $ ax run --new-netns=false curl -I 1.1.1.1:8000


Run the server without Docker
-----------------------------

Let's start another web server.  This time natively without a container.  We will assign it an IP address of 1.1.1.1 and give it a DNS resolvable name 'webserver'.
::

   $ sudo ax run --ip 1.1.1.1 --name webserver python -m SimpleHTTPServer 8000 &

Again we can curl the webserver using either the IP address or the name that we have assigned.  Since the previous server and this one are assigned the same IP address, incoming requests would get load balanced across the two.
::

   $ sudo ax run -- curl -I webserver:8000

or::

   $ sudo ax run -- curl -I 1.1.1.1:8000


Connecting to services without AppSwitch AKA 'Ingress Gateway'
--------------------------------------------------------------

Services run by AppSwitch can be exposed externally with the ``--expose`` option.
::

   $ sudo ax run --ip 1.1.1.1 --name webserver --expose 8000:192.168.0.10:10000 python -m SimpleHTTPServer 8000 &

We can now connect to the web server using curl without wrapping the command in ``ax``
::

   $ curl -I 192.168.0.10:10000
   HTTP/1.0 200 OK
   Server: SimpleHTTP/0.6 Python/2.7.5
   Date: Wed, 06 Jun 2018 03:18:02 GMT
   Content-type: text/html; charset=UTF-8
   Content-Length: 274

When starting the web server you can see we have explicitly specified the IP address of the node we wish to expose the service.  We could also have used the wildcard address ``0.0.0.0`` to expose the service on all nodes in the cluster.

If you start two web server processes both with the same name and IP then AppSwitch will load balance when a connection to that name/IP is made.


Virtual Services
~~~~~~~~~~~~~~~~

If we start the web server without an IP
::

   $ sudo ax run -- python -m SimpleHTTPServer 8000

And view the app
::

   $ ax get apps
                      NAME                    APPID    NODEID   CLUSTER     APPIP    DRIVER     LABELS          ZONES
   -----------------------------------------------------------------------------------------------------------------------
     <9142a421-00e8-483e-83d0-eea9716c849a>  f000015d  host    appswitch  10.0.2.15  user    zone=default  [zone==default]

We can associate an IP address with this app by creating a virtual service.
::

   $ ax create vservice --ip 1.1.1.1 --backends 10.0.2.15  --expose 8000:10000 myvsvc
   Service 'myvsvc' created successfully with IP '1.1.1.1'.
   $ ax get vservices
     VSNAME  VSTYPE   VSIP       VSPORTS      VSBACKENDIPS
   -------------------------------------------------------
     myvsvc  Random  1.1.1.1  [{8000 10000}]  [10.0.2.15]

Now we can curl to the virtual IP or the virtual name.  This feature
enables multiple IPs for the same server since the server is still
available at the IP assigned it by ax.  Furthermore, if we start more than
one server we can add them all as backends for the virtual service and
AppSwitch will load balance when connecting to the virtual name or IP.
Currently round-robin and random load balancing strategies are supported.
::

   $ sudo ax run -- curl -I myvsvc:8000
   $ sudo ax run -- curl -I 1.1.1.1:8000


Multi-Node AppSwitch Cluster
============================

Bring up a second VM so that we can build a 2 node AppSwitch cluster:
::

   $ vagrant up host01

ssh in and start the AppSwitch daemon
::

   $ vagrant ssh host01
   host01 $ docker-compose --file docker-compose-host01.yaml up -d

Now we can have a play with these two nodes. We can then try curl'ing the
server we brought up earlier on host00 from host01.
::

   host01 $ sudo ax run -- curl -I webserver:8000
   HTTP/1.0 200 OK
   Server: SimpleHTTP/0.6 Python/2.7.5
   Date: Wed, 06 Jun 2018 03:18:02 GMT
   Content-type: text/html; charset=UTF-8
   Content-Length: 274

Let's restart the webserver on host00, this time passing it the ingress gateway
wildcard address
::

   host00 $ sudo ax run --ip 1.1.1.1 --name webserver --expose 8000:0.0.0.0:10000 python -m SimpleHTTPServer 8000 &

From either host we can now directly curl the web server
::

   $ curl -I 192.168.0.11:10000
   $ curl -I 192.168.0.10:10000

   
Multi-Node Multi-Cluster AKA AppSwitch Federation
=================================================

Let us now configure two AppSwitch clusters each consisting of two nodes.  As documented in the Vagrant file, the topology looks as follows:
::

  #              cluster0                                          cluster1
  #     host01              host00                        host10              host11
  #
  #                        10.0.0.10  -----------------  10.0.0.11
  # 192.168.0.11  -----  192.168.0.10                  192.168.1.10  -----  192.168.1.11


Bring up the other two VMs:
::

   $ vagrant up host10
   $ vagrant up host11


Configure and Start the Daemon
------------------------------

To configure the nodes for federation connectivity we must configure two of
the nodes as federation gateways, we use ``host00`` and ``host10`` as the
gateways.  Also each node must be configured with a cluster name,
``cluster0`` or ``cluster1``.

Start the AppSwitch daemon on each node
::

   $ docker-compose --file docker-compose-$(hostname).yaml up -d


Now we can have a play with these four nodes.  First let's start a web
server on ``host11``
::

   $ sudo ax run --ip 2.2.2.2 python -m SimpleHTTPServer 8000 &

We can then try curl'ing this server from host01:
::

   $ sudo ax run -- curl -I 2.2.2.2:8000
   HTTP/1.0 200 OK
   Server: SimpleHTTP/0.6 Python/2.7.5
   Date: Wed, 06 Jun 2018 03:18:02 GMT
   Content-type: text/html; charset=UTF-8
   Content-Length: 274

In this case, the client is able to reach the server on the specified IP address (``2.2.2.2``) even though it is in a completely different network, which could have been somewhere in the cloud.  AppSwitch is able to flatten the network even across hybrid environments without complex tunneling etc.
