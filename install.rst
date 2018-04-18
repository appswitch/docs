=========================
Install and Configuration
=========================

.. _install:

Running the AppSwitch docker image will start the ``ax`` daemon and also copy
the binary onto the host at ``/usr/bin/ax``.  No further installation is
required.


Running the Daemon
==================

It can be convenient to run the AppSwitch daemon in a container using
``docker-compose``.  Here is an example docker-compose file
::

    version: '2.3'

    volumes:
      appswitch_logs:

    services:
      appswitch:
        image: appswitch/ax
        pid: "host"
        network_mode: "host"
        privileged: true
        volumes:
          - /usr/bin:/usr/bin
          - /dev:/dev
          - /var/run/appswitch:/var/run/appswitch
          - appswitch_logs:/var/log
        env_file:
          - ${AX_CONFIG_FILE:-ax.config}

      test:
        image: python:alpine
        entrypoint: /usr/bin/ax run --name test
        command: python -m http.server
        privileged: true
        networks: []
        volumes:
          - /usr/bin:/usr/bin
          - /var/run/appswitch:/var/run/appswitch
        depends_on:
          - appswitch


As indicated in this docker-compose file the AppSwitch daemon can be
passed a config file by either setting an environment variable
``AX_CONFIG_FILE`` or placing the configuration options in a file called
``ax.config`` in the same directory as the docker-compose file above.

(You only need the ``/dev`` volume if you are using the kernel driver.)

The configuration file consists of environment variable definitions.
::

    # AppSwitch configuration file template.
    #
    #  Copy to ax.config or set AX_CONFIG_FILE within your environment.

    # Syscall forwarding driver
    AX_DRIVER=user

    # Cluster config options.
    AX_NODE_INTERFACE=
    AX_NEIGHBORS=

    # Federation config options.
    AX_CLUSTER=
    AX_FEDERATION_GATEWAY_IP=
    AX_FEDERATION_GATEWAY_NEIGHBORS=

    # Additional options
    # APPSWITCH_OPTS=--clean --dns-domain=local.appswitch
    APPSWITCH_OPTS=--clean


Of course you can just run the daemon from the command line if you prefer
but then you will need to pass all configuration options on the command line.