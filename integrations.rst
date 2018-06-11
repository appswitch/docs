============
Integrations
============

.. _integrations:

This guide describes AppSwitch's integration with Kubernetes and Istio.

Kubernetes
==========

AppSwitch can serve as the network backend for Kubernetes.  In a Kubernetes cluster, AppSwitch can be deployed with the DaemonSet spec like below.  The only required customization before creating the DaemonSet is ``args``:  a comma-separated list of one or more of existing nodes in the AppSwitch cluster (Eg., ``args: ["--neighbors","<ip1>,<ip2>"]``).  For simplicity and availability, all nodes in the Kubernetes cluster shown by ``kubectl get nodes`` can be added.


::

	---
	apiVersion: apps/v1
	kind: DaemonSet
	metadata:
	  name: kube-appswitch
	  namespace: kube-system
	  labels:
	    k8s-app: appswitch
	spec:
	  selector:
	    matchLabels:
	      name: kube-appswitch
	  template:
	    metadata:
	      labels:
		name: kube-appswitch
	    spec:
	      hostNetwork: true
	      hostPID: true
	      containers:
	      - name: appswitch-daemon
		image: appswitch/ax
		args: ["--neighbors","<ip1>,<ip2>"]
		resources:
		  requests:
		    cpu: "200m"
		    memory: "200Mi"
		  limits:
		    cpu: "200m"
		    memory: "200Mi"
		securityContext:
		  privileged: true
		volumeMounts:
		- name: bin
		  mountPath: /usr/bin
		- name: sock
		  mountPath: /var/run/appswitch
	      volumes:
		- name: bin
		  hostPath:
		    path: /usr/bin
		- name: sock
		  hostPath:
		    path: /var/run/appswitch


Then the DaemonSet itself can be created as follows.


::

	$ kubectl create -f kube-daemonset.yaml


At this point, the Kubernetes cluster is primed to run applications based on AppSwitch.  The pod spec needs a few small changes to use AppSwitch.  This sample nginx pod spec illustrates those changes:


::

	apiVersion: v1
	kind: Pod
	metadata:
	  name: nginx
	spec:
	  containers:
	  - name: nginx
	    image: nginx
	    command: ['/usr/bin/ax','run','--ip','1.1.1.1',nginx, '-g', 'daemon off;']
	    ports:
	    - containerPort: 80
	    volumeMounts:
	      - name: bin
		mountPath: /usr/bin/ax
	      - name: run
		mountPath: /var/run/appswitch
	  volumes:
	    - name: bin
	      hostPath:
		path: /usr/bin/ax
	    - name: run
	      hostPath:
		path: /var/run/appswitch


This creates an nginx pod enabled to run with AppSwitch.  The changes from a typical nginx pod spec are:


- ``command`` is prefixed with ``ax`` and its parameters
- ``/usr/bin/ax`` and ``/var/run/appswitch`` host paths are mounted into the container


The application can be deployed to the cluster the normal way.


::

	$ kubectl create -f nginx.yaml


Once deployed, you can connect to this `nginx` pod from any node in the cluster as follows.


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


Istio
=====


AppSwitch is integrated with Istio to serve as a highly efficient dataplane through the AppSwitch Istio agent that consumes Pilot (XDS) API and conveys traffic management policies to AppSwitch.
