Consul Smartstack
=================

This repository contains a recipe for running a AirBnB
`Smartstack <http://nerds.airbnb.com/smartstack-service-discovery-cloud/>`_\ -like
infrastructure, but not using `Nerve <https://github.com/airbnb/nerve>`_ and
`Synapse <https://github.com/airbnb/synapse>`_, but instead using Hashicorp's
`Consul <https://consul.io/>`_ and
`Consul-template <https://github.com/hashicorp/consul-template>`_.

Summarized, Smartstack routes requests to services used by applications in a
runtime environment by running local `haproxy <http://www.haproxy.org/>`_
instances on each host that route tcp/udp traffic to these services.
Applications don't distinguish between local and remote services, instead
everything looks like a local service and routing traffic to an available
instance of said service is managed by a separate system. In AirBnB's case that
separate system is Zookeeper, Nerve and Synapse. In this implementation it's
Consul and Consul-template.

I have extracted this from
`my Saltstack configuration <https://github.com/jdelic/saltshaker>`_ where I
use it to push services out to nodes.


Run Smartstack with consul and consul-template
----------------------------------------------
Basic steps:

* run consul with ``-config-dir=/etc/consul/services.d``, see
  ``consul-template.service`` for an example systemd config file that I use.
* register services in consul from nodes by putting service definitions in
  ``/etc/consul/services.d`` or have Nomad /
  [docker registrator](https://github.com/gliderlabs/registrator) do it for
  you.
* have consul-template listen to the service catalog by having a Python
  script pose as a ``consul-template`` template using the
  ``{{services}}`` catalog, therefor getting rerendered every time a service
  gets added or removed
* Said Python script is rendered and then immediately executed by
  consul-template taking parameters and a Jinja template rerendering haproxy
  configurations from said Jinja template and then the Python script reloads
  or restarts haproxy
* The included haproxy config templates define a local proxy
  (Smartstack-like) for proxying internal services to your applications on
  ``127.0.0.1`` and
* an external haproxy that using the concept of consul service tags,
  interpreted by Python, to create a loadbalancer haproxy that terminates SSL
  and forwards traffic to apps registering themselves as consul services on
  nodes

I run 3 instances of this setup in parallel on my servers, as you can see in my
[Salt Smartstack setup](https://github.com/jdelic/saltshaker/tree/master/srv/salt/haproxy):

  * One that routes (micro)services and essential services on localhost
  * One that routes the same (micro)services and essential services on the
    local Docker bridge interface. This allows services managed by Nomad or,
    for example, Docker Swarm to also reach services living outside the cluster
    edges
  * One that routes internet-facing services to internal endpoints (a
    common loadbalancer)


Why not use consul-template directly for templating the haproxy configuration?
------------------------------------------------------------------------------
There are limits to Golang's (and by extension consul-template's) templating
language that make this infeasible. Much of the infrastructure in this
repository depends on using consul service tags to pass information from the
service definition in Consul to haproxy. Since consul-template's templating
language does not support setting variables or other constructs (and the
`developers don't want to change that <https://github.com/hashicorp/consul-template/issues/399>`_\ )
an intermediate Python script is a good solution to provide a more expressive
template language.

consul-template now provides "Scratch storage", which are template-local
variables. This impro


Command-line arguments
----------------------
``servicerenderer.ctmpl.py`` as its filename suggests is a consul-template
template that renders into a Python script. The resulting Python script is
meant to be executed by the ``command`` directive of consul-template's template
configuration. The script supports a number of command-line arguments modifying
its behaviour. It's purpose is to select a set of services from the full
services catalog in Consul, passed into it through consul-template, pass that
set into a Jinja2 template and execute a command.

Query Syntax
++++++++++++
Queries are expressed as a comma-separated list of criteria which consist of
keys (field names) from Consul's service catalog and values ((sub)strings or
regular expressions) that must match the field's value. Regular expressions
start with ``regex=``. Using ``--include`` or ``--exclude`` multiple times
allows to express boolean ``OR`` semantics.

Examples:

.. code-block:

    --include 'tags=smartstack:internal,name=regex=^xyz$'
    --exclude tags=udp
    --exclude ip=192.168.56.
    --include tags=mytag,tags=myothertag,port=2323


====================== =======================================================
Parameter              Description
====================== =======================================================
--add-all              Add all services to the selected set. This allows you
                       to subtract services from the full set.
--include [query]      Add services matching the query to the filtered set of
                       services that is passed into the template. (see Query
                       Syntax)
--exclude [query]      Remove services matching the query to t
--smartstack-localip   Set the {{localip}} template variable. (Default:
                       ``127.0.0.1``).
--open-iptables TYPE   Execute ``iptables`` commands for all services selected
                       by ``--has/--has-not/--match/--no-match`` and append
                       INPUT and OUTPUT rules between the IP specified by
                       ``--smartstack-localip`` and the network. TYPE can be
                       either ``plain`` or ``conntrack``. ``plain`` results
                       in *both* INPUT and OUTPUT rules from and to the local
                       IP. ``contrack`` will limit these connections to those
                       that have the ``NEW`` state, assuming that your firewall
                       is already configured to route ``RELATED`` packets.
--only-iptables        Do not render any template or execute any command, just
                       set up iptables for matched services.
-D, --define KEY=VAL   Set KEY to VAL in the rendered template.
-o, --output           Name the output file for the rendered result. Default:
                       ``stdout``.
-c, --command          The command to invoke after rendering the template.
                       Will be executed in a shell.
template               The only positional command-line argument: specifies
                       the Jinja2 template to render.
====================== =======================================================


Template directives
-------------------
When consul-template executes the Python script it renders a Jinja template. It
wraps the consul service catalog in ``SmartstackService`` objects which are
managed by ``SmartstackServiceContainer`` instances. Using these allows easily
chainable selectors for services and their entries in Consul.

Template context
++++++++++++++++
The default template context contains all key/value pairs defined on the
command-line via ``-D``. It also always contains the following variables:

================= ============================================================
Template variable Description
================= ============================================================
``localip``       The value passed to ``--smartstack-localip`` and the IP used
                  for all iptables operations.
``services``      The pre-filtered list of services (``--has``, ``--has-not``,
                  ``--match`` and ``--no-match`` already applied) list of
                  services from Consul's service catalog wrapped in a
                  ``SmartstackServiceContainer`` (see below).
================= ============================================================


SmartStackServiceContainer
++++++++++++++++++++++++++
Whenever a group of services is returned, they are wrapped in an instance of
``SmartstackServiceContainer``. This versatile class behaves like a ``dict``
when it represents a number of services grouped by a common property or it
behaves like a ``list`` when it represents an unfiltered number of services.
Each service is itself represented by an instance of ``SmartstackService``.

================== ===========================================================
Attribute          Description
================== ===========================================================
``.services``      Either a ``dict`` representing the groups of services split
                   into groups by ``.group_by()`` or ``.group_by_tagvalue()``
                   or a ``list`` of services.
``.all_services``  Always a list of all services this instance of
                   ``SmartstackServiceContainer`` started out with. You will
                   rarely access this directly, use ``.ungroup()`` instead.
``.grouped_by``    A list of values the services contained in this
                   ``SmartstackServiceContainer``instance have been sorted by,
                   one after the other.
``.group_by_type`` A list of the types of groupings used, one after the other.
                   Each grouping can be of type ``field`` or ``tag``.
``.filtered_to``   A list of the criteria leading to this group. In nested
                   ``SmartstackServiceContainer`` instances, the
                   ``.filtered_to`` attribute of a child container is
                   equivalent to the ``.grouped_by`` property of the container
                   it was created from.
================== ===========================================================

============================ =================================================
Method                       Description
============================ =================================================
``.ungroup()``               Returns an unfiltered/ungrouped top-level
                             ``SmartstackServiceContainer`` representing all
                             services. This allows you to undo all previous
                             calls to ``.group_by()`` and
                             ``.group_by_tagvalue()``.
``.value_set(f)``            Return a ``Set[str]`` of all values of *f* in the
                             Consul services contained in the current
                             container. Valid values of *f* are all fields
                             returned in the Consul service catalog.
``.tagvalue_set(f)``         Return a ``Set[str]`` of all tags in the list of
                             tags on a Consul service defnition for which
                             ``tagvalue.startswith(f) is True``.
``.group_by(f)``             Return a ``SmartstackServiceContainer`` instance
                             which represents a
                             ``Dict[str, SmartstackServiceContainer]`` where
                             each existing value of field *f* in the Consul
                             service catalog is a key resolving to a list-like
                             container of all services where ``f == key``.
``.group_by_tagvalue(part)`` Return a ``SmartstackServiceContainer`` instance
                             which represents a
                             ``Dict[str, SmartstackServiceContainer]`` where
                             the keys are all tag values that started with
                             *part* (with *part* cut off) and the value is a
                             list-like container containing all
                             ``SmartstackService`` instances having a tag
                             ``part+key``.
============================ =================================================

You will probably never have to use these methods, but I'll document them
anyway:

==================== =========================================================
Method               Description
==================== =========================================================
``.add(...)``        Add a service to a ``list``-like container (raises
                     ``ValueError`` on a ``dict``-like container.
``.iter_services()`` Return a generator to iterate over all
                     ``SmartstackService`` instances contained. ``__iter__()``
                     is also defined, so you'll need this rarely.
``.keys()``          Returns the keys of a ``dict``-like container.
``.items()``         Returns the items of a ``dict``-like container.
``.count()``         Returns the numer of SmartstackService instances in a
                     ``list``-like container and the number of keys in a
                     ``dict``-like container.
==================== =========================================================


SmartStackService
+++++++++++++++++
Each individual Consul service is wrapped in a ``SmartstackService`` instance.

================== ===========================================================
Attribute          Description
================== ===========================================================
``.svc``           The "service dictionary". This is the deserialized JSON
                   structure returned by Consul for each service from the
                   Consul service catalog. This gives you direct access to all
                   data from Consul.
``.ip``            The service's IP address as defined in the Consul service
                   definition.
``.port``          The service's IP port as defined via the
                   ``smartstack:port:*`` tag *or* if that is not defined, the
                   service's IP port from its Consul service definition.
``.name``          The name of the service as defined in the Consul service
                   definition.
``.tags``          Returns a ``List[str]`` of all tags defined for this
                   service in the Consul service definition.
================== ===========================================================

==================== =========================================================
Method               Description
==================== =========================================================
``.tagvalue(part)``  If the service has a tag starting with *part*, returns
                     the tag with *part* cut off.
==================== =========================================================


Examples
--------
Look at the included haproxy configuration templates for example code.

* ``haproxy-external.jinja.cfg`` is a configuration template for a HTTP(S)
  loadbalancer supporting tag-based configuration for SNI and HTTP
  hostname-based backend routing.

* ``haproxy-internal.jinja.cfg`` is a configuration template for running a
  Smartstack infrastructure on every node in a cluster routing internal
  services from ``localhost`` on predefined ports, thereby allowing
  applications to be ignorant of where the services they are using are
  running.

* ``servicerenderer-internal.conf`` a consul-template configuration example.


Predefined Consul service tags
++++++++++++++++++++++++++++++
The example templates use a number of tags to configure basic attributes of
Smartstack and the external loadbalancer role.

=========================== ==================================================
Tag                         Description
=========================== ==================================================
smartstack:mode:TYPE        The haproxy mode to use for this service. Can be
                            any haproxy supported mode. Default: ``tcp``.
smartstack:port:PORT        An optional override for the service's IP port.
smartstack:protocol:PROT    Used to configure the external load balancer role.
                            Can be ``http`` or ``https`` or ``sni`` depending
                            on the internet-facing service. ``https`` will
                            terminate SSL on the loadbalancer, whereas ``sni``
                            can be used to send SSL traffic directly to the
                            backend and terminate it there. (loadbalancer only)
smartstack:https-redirect   A tag that creates a haproxy rule to redirect
                            a request over HTTP to HTTPS (loadbalancer only)
smartstack:hostname:HOST    Attaches an internet-facing service to the
                            hostname HOST via the HTTP Host header or SNI.
smartstack:internal         Marks services used for Smartstack configuration
                            via ``haproxy-internal.jinja.cfg``.
smartstack:external         Marks services that are hooked to to the external
                            load balancer via ``haproxy-external.jinja.cfg``.
haproxy:frontend:option:OPT Allows passing *OPT* to haproxy's *option* config.
haproxy:frontend:port:PORT  Forces haproxy to listen on PORT while sending
                            traffic to the service's port from Consul. This
                            allows you to fix frontend ports for dynamically
                            assigned backend ports (like Nomad and other
                            cluster schedulers use).
crt:CERT                    Adds *CERT* as a SSL certificate to the
                            loadbalancer haproxy in
                            ``haproxy-external.jinja.cfg`` so it can do SNI
                            and SSL termination.
=========================== ==================================================


License
=======

Copyright (c) 2017, Jonas Maurus
All rights reserved.

Redistribution and use in source and binary forms, with or without
modification, are permitted provided that the following conditions are met:

1. Redistributions of source code must retain the above copyright notice, this
   list of conditions and the following disclaimer.

2. Redistributions in binary form must reproduce the above copyright notice,
   this list of conditions and the following disclaimer in the documentation
   and/or other materials provided with the distribution.

3. Neither the name of the copyright holder nor the names of its contributors
   may be used to endorse or promote products derived from this software
   without specific prior written permission.

THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS "AS IS" AND
ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE IMPLIED
WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE ARE
DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT HOLDER OR CONTRIBUTORS BE LIABLE
FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR CONSEQUENTIAL
DAMAGES (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR
SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS INTERRUPTION) HOWEVER
CAUSED AND ON ANY THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY,
OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE
OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.
