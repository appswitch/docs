==============================
Installation and Configuration
==============================

.. _install:

AppSwitch client and daemon are both built as a static binary, ``ax``, with no external dependencies.  It is packaged into a docker image for convenience.  Installation of the binary (copying to /usr/bin) and bringing up of the daemon can be done by running the following comamnd:
::

    curl -L http://appswitch.io/docker-compose.yaml | docker-compose -f - up -d


It runs the latest release of AppSwitch docker image through a docker-compose file.  The compose file includes most common configuration options as environment variables.  Additional options can be passed through AX_OPTS.
::

    #
    # docker-compose file AppSwitch Daemon
    #
    # Bring up with:
    #  docker-compose up -d appswitch
    #

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
        environment:
          - AX_DRIVER=user # Syscall forwarding driver
          - AX_NODE_INTERFACE= # Node interface to use by daemon.  Accepts IP address or interface name, eg eth0
          - AX_NEIGHBORS= # List (csv) of IP addresses of cluster neighbors
          - AX_CLUSTER= # Cluster name.  Required if cluster is part of a federation
          - AX_FEDERATION_GATEWAY_IP= # IP address or `interface` name for federation connectivity
          - AX_FEDERATION_GATEWAY_NEIGHBORS= # List (`csv`) of IP addresses of federation neighbors (other gateway nodes)
          - AX_OPTS=--clean # Remove any saved state from previous sessions


