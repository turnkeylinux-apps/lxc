Usage
-----

Creating a TurnKey LXC container is done by specifying ``turnkey`` as
the template when invoking ``lxc-create``, for example::

    # lxc-create -n CONTAINER_NAME -t turnkey

The TurnKey LXC template has required and optional arguments.
``lxc-create`` will pass the arguments specified after a double dash to
the template. For example, to show the templates usage we could run::

    # lxc-create -n CONTAINER_NAME -t turnkey -- --help

    TurnKey LXC Template Syntax: appname [options]

    Arguments::

        appname          Appliance name (e.g., core)

    Options::

        -a --arch=       Appliance architecture (default: hosts architecture)
        -v --version=    Appliance version (default: 13.0-wheezy)
        -c --cachedir=   Path to appliance cache (default: /var/cache/lxc/turnkey)
        -x --aptproxy=   Address of APT Proxy (e.g., http://192.168.121.1:3142)

        -i --inithooks=  Path to inithooks.conf (e.g., /root/inithooks.conf)
                         Reference: https://www.turnkeylinux.org/docs/inithooks

        -l --netlink=    Value of lxc.network.link (default: br0)
                         Specify none to omit network configuration

    Example usage::

        lxc-create -n core -t turnkey -- core -i /root/inithooks.conf

Inithooks (preseeding)
----------------------

An `inithooks`_ configuration must be specified in order to preseed the
appliance on firstboot. For example, lets create an inithooks
configuration for the TurnKey Wordpress appliance::

    # cat > /root/inithooks.conf <<EOF
    export ROOT_PASS=secretrootpass
    export DB_PASS=secretmysqlpass
    export APP_PASS=secretadminwppass
    export APP_EMAIL=admin@example.com
    export APP_DOMAIN=www.example.com
    export HUB_APIKEY=SKIP
    export SEC_UPDATES=FORCE
    EOF

Note, an example inithooks configuration is included in the TurnKey LXC
appliance in the /root directory for convenience.

Networking (bridged vs. NAT)
----------------------------

TurnKey LXC supports two networking configurations out of the box,
bridged and NAT.

Bridged (br0)
'''''''''''''

When TurnKey LXC is deployed on bare-metal, the easiest configuration
would be bridged, in which case the container would be allocated an IP
address on the network (via DHCP), and all its services (e.g., SSH, Web
server, etc.) would be accessible directly.

NAT (natbr0)
''''''''''''

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

Creating a container (wordpress, NAT)
-------------------------------------

Continuing from the earlier inithooks example, we'll create a TurnKey
Wordpress container. We'll also use the NAT bridge as it requires some
extra steps to expose the containers services to the network.

Additionally, we'll also specify an APT proxy (preconfigured on the 
TurnKey LXC appliance) so other containers can leverage the cache.

1. Create the container::

    # lxc-create -n wp1 -t turnkey -- wordpress -i /root/inithooks.conf -l natbr0 -x http://192.168.121.1:3142

2. Start the container in the background::

    # lxc-start -n wp1 -d

3. Expose the containers web services to the network by creating an
   nginx site configuration to proxy all web requests (ports 80, 443,
   12320, 12321, 12322) destined for www.example.com to the container on
   the corresponding ports::

    # nginx-proxy www.example.com wp1

4. Expose the containers SSH service to the network by configuring
   iptables on the host to forward the traffic it recieves on port 2222
   to the container on port 22::

    # host wp1
    wp1 has address 192.168.121.165

    # iptables-nat add 2222 192.168.121.165:22


.. _inithooks: https://www.turnkeylinux.org/docs/inithooks

