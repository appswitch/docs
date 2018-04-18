======================
AppSwitch Architecture
======================

.. _arch:

An application that wishes to communicate with other applications over the
network must be associated with a network endpoint.  Under the OSI model
this means a transport layer address (consisting of a port number and a
protocol) and a network layer address (for example an IPv4 address).
Traditionally this network endpoint is associated with an internal endpoint
(a network socket).

AppSwitch removes this connection between the network endpoint and the
internal endpoint.  The network endpoint seen by the application then
becomes 'virtual' allowing administrators to use any network address.  This
means that an application can be given any IPv4 address and any port number
and AppSwitch will connect this to an internal endpoint and handle the
translation.

AppSwitch integrates with the Domain Name System.  Applications can be
given any name and AppSwitch will facilitate that name being resolved using
DNS.

AppSwitch does not effect the network data path (i.e reads and writes to
the socket).  For this reason data throughput is maintained and in some
cases even improved (see performance comparison between Linux bridge and
AppSwitch).


.. _hierarchy:

AppSwitch Hierarchical Model
============================

AppSwitch model has the following hierarchy

Federation:

- Contains a set of clusters.

Cluster:

- Contains a set of nodes with mutual IP connectivity.
- A cluster may have one or more federation gateways.
- Federation gateways across clusters peer with each other over a gossip
  channel.

Node:

- Contains applications managed by AppSwitch.
- Each node runs an AppSwitch daemon instance.

Application:

- Contains zero or more client / server sockets, all maintained by
  the AppSwitch daemon on behalf of the application.


.. _servicetable:

Service Table
=============

AppSwitch provides a logically centralized data structure called *service table*
that maintains a record of all currently running services across the cluster.
The table is automatically updated as services come and go.

When an application calls listen() system call, a new service is automatically
added to the service table along with a set of system and user-defined
attributes.  The service attributes include app-id, name (if specified),
protocol, app-ip:app-port, host-ip:host-port and labels.

Clients running in an ax cluster would be able to access the services listed in
the server table by simply connect()ing to the app-ip:app-port listed in the
service table entry.  AppSwitch transparently performs service discovery, access
control and traffic management based on specified policy and the contents of the
service table and appropriately directs the connection.


.. _ingress:

Connectivity with External Entities
===================================

Flexible ingress, egress, and federation capabilities are provided for
applications running on an AppSwitch cluster to communicate with non-AppSwitch
external entities or to communicate across clusters.


Ports
=====

AppSwitch uses (by default) the following ports

| 6664 :ref:`rest-port-label`
| 7946 Cluster gossip protocol (see :ref:`serf-label`)
| 7947 Federation gossip (see :ref:`federation-label`)
| 6660 Ingress federation proxy (see :ref:`federation-label`)
|

These port numbers are configurable, please see links above for details.

