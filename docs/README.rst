Usage
-----

Creating a TurnKey LXC container is done by specifying ``turnkey`` as the
template when invoking ``lxc-create``, for example::

    # lxc-create -n CONTAINER_NAME -t turnkey

The TurnKey LXC template has required and optional arguments. ``lxc-create``
will pass the arguments specified after a double dash to the template. For
example, to show the templates usage we could run::

    # lxc-create -n CONTAINER_NAME -t turnkey -- --help

    TurnKey LXC Template Syntax: appname [options]

    Arguments::

        appname          Appliance name (e.g., core)

    Options::

        -a --arch=       Appliance architecture (default: hosts architecture)
        -v --version=    Appliance version (default: 13.0-wheezy)
        -c --cachedir=   Path to appliance cache (default: /var/cache/lxc/turnkey)

        -i --inithooks=  Path to inithooks.conf (e.g., /root/inithooks.conf)
                         Reference: http://www.turnkeylinux.org/docs/inithooks

        -l --netlink=    Value of lxc.network.link (default: br0)
                         Specify none to omit network configuration

    Example usage::

        lxc-create -n core -t turnkey -- core -i /root/inithooks.conf

Inithooks (preseeding)
----------------------

An `inithooks`_ configuration must be specified in order to preseed the
appliance on firstboot. For example, lets create an inithooks configuration for
the TurnKey Wordpress appliance::

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

TurnKey LXC supports two networking configurations out of the box, bridged
(br0) and NAT (natbr0).

Bridged
'''''''

When TurnKey LXC is deployed on baremetal, the easiest configuration would be
bridged, in which case the container would be allocated an IP address on the
network (via DHCP), and all its services (e.g., SSH, Web server, etc.) would be
accessible directly.

NAT
'''

On the other hand, when deployed in a virtual environment (e.g., VirtualBox)
which itself is using a bridged interface, bridging a bridge leads to issues as
the VE's bridge will ignore packets destined for the VM's bridge. It is
possible to workaround this, but the easier solution is to use the NAT bridge
interface.

In this scenario, the container will be allocated an internal IP address, and
will be accessible from the host, but not the network. In order to expose the
containers services to the network, we can configure the host to forward
traffic it receives on a port X to the container on a port Y. This is achieved
via iptables.

Creating a container (wordpress, NAT)
-------------------------------------

Continuing from the earlier inithooks example, we'll create a TurnKey Wordpress
container. We'll also use the NAT bridge as it requires an extra step.

1. Create the container::

    # lxc-create -n blog1 -t turnkey -- wordpress -i /root/inithooks.conf -l natbr0

2. Start the container in the background::

    # lxc-start -n blog1 -d

3. Expose some of the containers services to the network (SSH, HTTP/S)::

    # host blog1
    blog1 has address 192.168.121.165

    # iptables-nat add 2222 192.168.121.165:22

    # iptables-nat add 80 192.168.121.165:80
    # iptables-nat add 443 192.168.121.165:443

Now, all traffic destined for the TurnKey LXC host on port 2222 will be
forwarded to the container (port 22), so we could connect over the network to
the container using SSH::

    # ssh -p 2222 root@host_ip

Additionally, HTTP/S traffic destined for the host's IP address (port 80/443)
will be forwarded to the containers web server (port 80/443).

.. _inithooks: http://www.turnkeylinux.org/docs/inithooks

