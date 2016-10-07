Run Smartstack with consul and consul-template
==============================================

  * run consul
  * register services in consul from nodes by putting service definitions in `/etc/consul/services.d`
  * have consul-template listen to the service catalog by having a Python script pose as a `consul-template` template 
    using the `{{services}}` catalog, therefor getting rerendered every time a service gets added or removed
  * Said Python script is rendered and then immediately executed by consul-template taking parameters and a Jinja template
    rerendering haproxy configurations from said Jinja template and then the Python script reloads/restarts haproxy
  * The included haproxy config templates define a local proxy (Smartstack-like) for proxying internal services to your
    applications on `127.0.0.1` and
  * an external haproxy that using the concept of consul service tags, interpreted by Python, to create a loadbalancer
    haproxy that terminates SSL and forwards traffic to apps registering themselves as consul services on nodes