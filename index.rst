AppSwitch Documentation
=======================

AppSwitch performs service discovery, access control and traffic
management functions on behalf of the applications by transparently
taking over the applications' network API calls.  In addition, AppSwitch
decouples the application from the constructs of underlying network
infrastructure by projecting a consistent, virtual view of the network to
the application.  In abstract terms, it combines the
application-level functionality offered by the service mesh approach
with the familiarity and compatibility of traditional networking.

Some of the use cases include:

* Automatically curated service registry for seamless service discovery
  even across hybrid environments.

* Extremely efficient enforcement of label-based access controls without
  any data path processing.

* Proxy-less load-balancing and traffic management across service instances.

* IP address portability with ability to assign arbitrary IP addresses to
  applications regardless of underlying network.

* Transparently redirect connection requests to alternate services without
  using NAT in case of application migration.

* "flat" connectivity with client IP preservation across hybrid network
  environments.


.. toctree::
   :maxdepth: 2
   :caption: Contents:

   overview
   arch
   install
   cli
   integrations
   reading

Indices and tables
==================

* :ref:`genindex`
* :ref:`modindex`
* :ref:`search`
