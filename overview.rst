=========================
Functionality Overview
=========================

.. _overview:

This section walks through a quick demo of AppSwitch functionality.

You can specify any IP address and name you want when bringing up an
application.  Like this
::

    $ ax run --ip 1.1.1.1 nginx -g 'daemon off;'

That brings up ``nginx`` server on 1.1.1.1, no matter the IP address of the
host.  Note that no interfaces or other network artifacts are created on the
host for this to work.

(Please note, AppSwitch currently only supports IPv4.  nginx should be
configured to listen on IPv4 address only.)

Internally appSwitch runs the application (nginx in this case) in an empty
network namespace with no devices and tracks its network API calls through an
equivalent of FUSE for network.

You can ``curl`` the server as follows
::

    $ ax run --ip 2.2.2.2 curl -I 1.1.1.1
    HTTP/1.1 200 OK
    Server: nginx/1.12.2
    Date: Tue, 20 Mar 2018 21:21:22 GMT
    Content-Type: text/html
    Content-Length: 3700
    Last-Modified: Wed, 18 Oct 2017 08:08:18 GMT
    Connection: keep-alive
    ETag: "59e70bf2-e74"
    X-Backend-Server: host2
    Accept-Ranges: bytes

Here curl is given a different IP address.  But since it's only acting as a
client, you can do without an IP.  So this works just as well
::

    $ ax run --curl -I 1.1.1.1


Works with names as well
------------------------

The server can be given a DNS name rather than or in addition to an IP address.
Once again, no other changes at the infrastructure level are needed for this to
work.  The point of is to remove unnecessary interactions between applications
and infrastructure.
::

    $ ax run --name web nginx -g 'daemon off;'

Then you can connect using the name and have ``ax`` handle the DNS
::

    $ ax run -- curl -I web

AppSwitch internally makes up an IP address for the purposes of API
compatibility but the clients can simply refer to the service by its name
without ever knowing about the internal IP address.


Simple load balancing
---------------------

You can also bring up another server and give it the same IP address.  Don't
worry, that won't cause IP or port conflict.  Client connections would be
automatically load balanced across those servers.

It also works just fine even if the application runs within a container.  Just
need to pass a couple extra options to "mount" appswitch into the container.  In
fact, it doesn't matter where the application runs.  It could be bare metal, VM,
container or somewhere in the cloud as long as there is some type of
connectivity underneath.
::

    $ docker run -it \
        --net none \
        -v /var/run/appswitch:/var/run/appswitch \
        -v /usr/bin/ax:/usr/bin/ax \
        --entrypoint /usr/bin/ax \
        --cap-add NET_ADMIN \
        --cap-add SYS_ADMIN \
        run --ip 1.1.1.1 nginx -g 'daemon off;'

Note that it's an off-the-shelf Docker image with no customizations etc.  Also
notice ``--net none`` option.  As far as Docker is concerned, the container has
no network connectivity.  ``nginx`` gets its network through AppSwitch.

Once the second nginx server is started (as a Docker container, in this case),
the client connections would be automatically load balanced across those two
instances.  You could observe it using ``watch``
::

    $ watch ax run -- curl -I 1.1.1.1


Security grouping through intuitive labels
------------------------------------------

You can attach labels to applications while bringing them up under AppSwitch.
Among other things, labels are used to enforce isolation.  Something like
::

    $ ax run --ip 1.1.1.1 --labels role=prod nginx -g 'daemon off;'&
    $ ax run --ip 1.1.1.1 --labels role=test nginx -g 'daemon off;'&

You can verify that there are multiple servers serving on the same IP address by
listing the contents of the *service table* with ``ax`` command
::

    $ ax show srtables servers
    NODEID    CLUSTER     APPID     PROTO   SERVICEADDR      IPV4ADDR
    ------------------------------------------------------------------------
    host0   appswitch     f83e8600    tcp    1.1.1.1:80     10.0.2.15:40010
    host0   appswitch     f880063e    tcp    1.1.1.1:80     10.0.2.15:40019

At this point, clients only get load balanced to servers that match the labels.
For example, the following client with label ``role=test`` will only connect to
the ``role=test`` server above but never to ``role=prod`` server even though
both carry the same IP address.
::

    $ ax --labels role=test curl -I 1.1.1.1

A naked client without any labels would not be able access either server.

