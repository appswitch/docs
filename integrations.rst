============
Integrations
============

AppSwitch is integrated with Kubernetes as a CNI plugin and a daemonset.  AppSwitch is also integrated with Istio to serve as its dataplane through an agent that consumes Pilot (XDS) API and conveys traffic management policies to AppSwitch.

.. _integrations:

This guide describes the architecture of AppSwitch's integration with Kubernetes and Istio.

Kubernetes
==========

AppSwitch can serve as the network backend for Kubernetes.

Istio
=====

AppSwitch can serve as an efficient dataplane for Istio.
