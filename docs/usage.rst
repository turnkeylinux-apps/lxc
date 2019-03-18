Usage
-----

Creating a TurnKey LXC container is done by specifying ``turnkey`` as
the template when invoking ``lxc-create``, for example::

    # lxc-create -n CONTAINER_NAME -f CONFIG_FILE -t turnkey -- APPNAME [template options]

The TurnKey LXC template has required and optional arguments.
``lxc-create`` will pass the arguments specified after a double dash to
the template. For example, to show the templates usage we could run::

    # lxc-create -n CONTAINER_NAME -t turnkey  -- --help

    TurnKey LXC Template Syntax: appname [options]

    Arguments::

        appname          Appliance name (e.g., core)

    Options::

        -a --arch=       Appliance architecture (default: hosts architecture)
        -v --version=    Appliance version (default: latest available)
        -x --aptproxy=   Address of APT Proxy (default: internal apt-cacher-ng)

        -i --inithooks=  Path to inithooks.conf (default: /root/inithooks.conf)
                         Reference: https://www.turnkeylinux.org/docs/inithooks

           --rootfs=     Path to root filesystem (default: $path/$name/rootfs)
        -c --clean       Clean the cache i.e. purge all downloaded appliance images

    Required options (passed automatically by lxc-create):

        -n --name=       container name ($name)
        -p --path=       container path_name ($path/$name)

    Example usage::

        lxc-create -n core -f /etc/lxc/bridge.conf -t turnkey -- core -i /root/inithooks.conf

.. note:: This document uses the domain **example.com** in accordance with IETF `RFC-2606`_. Please substitute a domain name appropriate for your local environment.

Inithooks (preseeding)
----------------------

An `inithooks`_ configuration must be specified in order to preseed the
appliance on firstboot. For example, lets create an inithooks
configuration for the TurnKey Wordpress appliance::

    # cat > /root/wp.inithooks.conf <<EOF
    export ROOT_PASS=secretrootpass
    export DB_PASS=secretmysqlpass
    export APP_PASS=secretadminwppass
    export APP_EMAIL=admin@example.com
    export APP_DOMAIN=www.example.com
    export HUB_APIKEY=SKIP
    export SEC_ALERTS=SKIP
    export SEC_UPDATES=FORCE
    EOF

If you wish to preseed static IP addresses for a `bridged` container, include the following lines in the `wp.inithooks.conf` file. ::

    export IP_CONFIG=static
    export IP_ADDRESS=XX.XX.XX.XX     # your static ip
    export IP_NETMASK=255.255.255.0   # your netmask
    export IP_GW=YY.YY.YY.YY          # your gateway address
    #export IP_DNS1=DD.DD.DD.DD       # optional first dns server address
    #export IP_DNS2=EE.EE.EE.EE       # optional second dns server address

.. note:: An example inithooks configuration is included in the TurnKey LXC appliance in the /root directory for convenience. Be sure to change the domain, passwords, and addresses to suit your environment.  Uncomment the last two lines if you want to specify the optional domain name servers.

Networking (bridged vs. NAT)
----------------------------

TurnKey LXC supports two networking configurations out of the box,
bridged and NAT. The TurnKey LXC template follows the convention established by
other templates and lets lxc-create control the network configuration through
the '-f <config_file>' option. For convenience, two preconfigured files are
provided; `/etc/lxc/bridge.conf` and `/etc/lxc/natbridge.conf`. The NAT bridge
configuration is now the default, when no config file is specified.

Bridged (br0)
'''''''''''''

When TurnKey LXC is deployed on bare-metal, the easiest configuration
would be bridged, in which case the container would be allocated an IP
address on the network (via DHCP), and all its services (e.g., SSH, Web
server, etc.) would be accessible directly.

NAT (natbr0) (default)
''''''''''''''''''''''

On the other hand, when deployed in a virtual environment (e.g.,
VirtualBox) which itself is using a bridged interface, bridging a bridge
leads to issues as the VE's bridge will ignore packets destined for the
VM's bridge. It is possible to workaround this, but the easier solution
is to use the NAT bridge interface.

Similarly, when deploying to an environment that is already NAT'ed, such
as Amazon EC2, the VM won't be able to request additional IP addresses,
so using the NAT bridge interface is required.

When using the NAT bridge, the container will be allocated an internal
IP address from the host (dnsmasq), and will be accessible from the host
itself, but not the network. In order to expose the containers services
to the network, the host can be configured to proxy and/or forward
traffic to the container based on a set of rules.

Usage: nginx-proxy
''''''''''''''''''

The current version of nginx-proxy supports the v14.x & v15.x appliances and is
decoupled from lxc so it can proxy any upstream vm or container. Options
exist which make it easier to cleanup when containers are removed and to better
support the Ansible appliance. Templates now use the Jinja2 style, although
Jinja2 is not yet used to render the output files. This feature may be added in
future versions. ::

    nginx-proxy version 2.3: GNU General Public License version 3
    Create site configuration to proxy requests destined for domain to host

    Syntax: nginx-proxy [-d|--domain] domain [-n|--name] host [-r|--remove]

    Arguments::

        domain           source domain (fqdn)
        host             destination host name

    Options::

        -h --help        usage: display this message
        -d --domain      source domain
        -n --name        host name
        -l --list        list domains and hosts
        -r --remove      remove a proxy from domain(s) to host
        -t --template    use alternate template
        -c --check       indicate if any changes would be made

    Examples::

        # create a proxy from domain 'www.example.com' to host 'wordpress' 
        nginx-proxy --domain www.example.com --name wordpress

        # remove a proxy from domain 'www.example.com' to host 'wordpress'
        nginx-proxy --remove -d www.example.com -n wordpress

        # remove all proxies for host 'wordpress'
        nginx-proxy --remove -d all -n wordpress

        # run in check-mode making no changes, but indicating what would be changed
        nginx-proxy --check -d www.example.com -n wordpress

    Exit Codes::

            0    no changes were made or would have been made (check-mode)
            1    changes were made or would have been made (check-mode)
            2    fatal error prevented command completion

    Notes::

        # also supports the v13.0 syntax
        nginx-proxy www.example.com wordpress

        # uses Jinja2 style templates for variable substitution
        # default template (preconfigured for ports 80, 443)
        /etc/nginx/templates/default.j2

        # lxc template (preconfigured for ports 80, 443, 12320, 12321, 12322)
        /etc/nginx/templates/container.j2

Usage: iptables-nat
'''''''''''''''''''
::

    Syntax: iptables-nat action s_port d_addr:d_port
    Add or delete iptables nat configurations

    Arguments::

        action          action to perform (add|del|info)
        s_port          source port on host
        d_addr:d_port   destination ip address and port

    Examples::

        iptables-nat add 2222 192.168.121.150:22
        iptables-nat del 2222 192.168.121.150:22


Creating a container (wordpress, bridged)
-----------------------------------------

Continuing from the earlier inithooks example, we'll create a TurnKey
Wordpress container using the bridged network configuration.

1. Create the container::

    # lxc-create -n wp1 -f /etc/lxc/bridged.conf -t turnkey -- wordpress -i /root/wp.inithooks.conf -v 15.0-stretch

    This could have been shortened because the version now defaults to `latest available`.:

    # lxc-create -n wp1 -f /etc/lxc/bridged.conf -t turnkey -- wordpress -i /root/wp.inithooks.conf

2. Start the container::

    # lxc-start -n wp1

3. List the containers::

    # lxc-ls -f

Creating a container (wordpress, NAT)
-------------------------------------

Now we'll create a second TurnKey Wordpress container.
We'll also use the NAT bridge as it requires some
extra steps to expose the containers services to the network.

1. Create the container::

    # lxc-create -n wp2 -f /etc/lxc/natbridge.conf -t turnkey -- wordpress -i /root/wp.inithooks.conf

    This could have been shortened because natbridge.conf is the default config:

    # lxc-create -n wp2 -t turnkey -- wordpress -i /root/wp.inithooks.conf


2. Start the container::

    # lxc-start -n wp2

3. Expose the containers web services to the network by creating an
   nginx site configuration to proxy all web requests (ports 80, 443,
   12320, 12321, 12322) destined for www.example.com to the container on
   the corresponding ports::

    # nginx-proxy --domain www.example.com --name wp2

4. Expose the containers SSH service to the network by configuring
   iptables on the host to forward the traffic it receives on port 2222
   to the container on port 22::

    # host wp2
    wp2 has address 192.168.121.165

    # iptables-nat add 2222 192.168.121.165:22

Removing a container (wordpress, NAT)
-------------------------------------

Now we'll remove the container, wp2, we just created.

1. Stop the proxy from forwarding requests to the container::

    # nginx-proxy --remove -d www.example.com -n wp2

   Note that both domain and container name must be specified when
   removing a proxy. This is because multiple domains may be forwarded
   to the same container.

2. Remove the iptables NAT::

    # iptables-nat del 2222 192.168.121.165:22

3. Stop the container::

    # lxc-stop -k -n wp2

    The kill option [-k] is optional and usually unnecessary.

4. Destroy the container::

    # lxc-destroy -n wp2

   or combine steps three and four::

    # lxc-destroy -f -n wp2

Apt Caching Proxy
-----------------

The LXC appliance uses ``apt-cacher-ng`` listening on ``port 3142`` for a caching
proxy server.  All containers are now configured by default to use the internal
cache (no longer necessary to include the ``-x`` option on the command line).

In some circumstances, it is desirable to use an external apt proxy.  For example,
a small development shop with several developer workstations, a TKLdev appliance
for building apps, an LXC appliance for testing, and other TurnKey appliances
for various stages of development and production.  To conserve bandwidth, we want
to have all workstations and appliances share a common apt proxy.

When an external apt proxy is available, the LXC appliance will continue to configure
all containers to use the internal ``apt-cacher-ng`` cache which will now forward
the request to the external apt proxy.  This can be configured in one of two ways.

1. If you are using pre-seeding, you can add the ``url`` of the external apt cache
   to the ``inithooks.conf`` file::

    export APT_PROXY=http://[external_proxy_host_domain||external_proxy_ip]:[port]

   Note that the ``url`` must be compatible with ``apt``'s proxy specification.

2. In all other cases, you can add the export line above to ``/root/.bashrc.d/apt-proxy``
   and then restart the appliance.  You can use this method if you forgot to
   pre-seed, or if you want to change the external apt cache.


.. _inithooks: https://www.turnkeylinux.org/docs/inithooks
.. _RFC-2606:   https://tools.ietf.org/html/rfc2606

