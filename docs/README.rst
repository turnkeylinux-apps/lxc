Usage
-----

Creating a TurnKey LXC container is done by specifying ``turnkey`` as
the template when invoking ``lxc-create``, for example::

    # lxc-create -n CONTAINER_NAME -f CONFIG_FILE -t turnkey

The TurnKey LXC template has required and optional arguments.
``lxc-create`` will pass the arguments specified after a double dash to
the template. For example, to show the templates usage we could run::

    # lxc-create -n CONTAINER_NAME -t turnkey  -- --help

    TurnKey LXC Template Syntax: appname [options]

    Arguments::

        appname          Appliance name (e.g., core)

    Options::

        -a --arch=       Appliance architecture (default: hosts architecture)
        -v --version=    Appliance version (default: 14.0-jessie)
        -x --aptproxy=   Address of APT Proxy (e.g., http://192.168.121.1:3142)

        -i --inithooks=  Path to inithooks.conf (default: /root/inithooks.conf)
                         Reference: http://www.turnkeylinux.org/docs/inithooks

           --rootfs=     Path to root filesystem (default: $path/$name/rootfs)
        -c --clean       Clean the cache i.e. purge all downloaded appliance images

    Required options (passed automatically by lxc-create):

        -n --name=       container name ($name)
        -p --path=       container path_name ($path/$name)

    Example usage::

        lxc-create -n core -f /etc/lxc/bridge.conf -t turnkey -- core -i /root/inithooks.conf

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
bridged and NAT. Starting in v14.0, the TurnKey LXC template follows the
convention established by other templates and lets lxc-create control
the network configuration through the '-f <config_file>' option.
For convenience, two preconfigured files are provided, /etc/lxc/bridge.conf,
and /etc/lxc/natbridge.conf. The NAT bridge configuration is now the default,
when no config file is specified.

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

The 2.0 version of nginx-proxy has been updated to support the v14.0
appliances and using lvm as a backing store. Actually any form of backing
store should work, but only dir and lvm have been tested. Some new options
have been added to make it easier to cleanup when containers are removed
and to better support the Ansible appliance. ::

    nginx-proxy version 2.0: GNU General Public License version 3
    Create site configuration to proxy requests destined for domain to container
    
    Syntax: nginx-proxy [-d|--domain] domain [-n|--name] container [-r|--remove]
    
    Arguments::
    
        domain           source domain
        container        destination container name
    
    Options::
    
        -h --help        usage: display this message
        -d --domain      source domain
        -n --name        container name
        -l --list        list domains and containers
        -r --remove      remove a proxy from domain to container
        -c --check       indicate if any changes would be made
    
    Examples::
    
        # create a proxy from domain 'www.example.com' to container 'wordpress' 
        nginx-proxy --domain www.example.com --name wordpress
    
        # remove a proxy from domain 'www.example.com' to container 'wordpress'
        nginx-proxy --remove -d www.example.com -n wordpress
    
        # run in check-mode making no changes, but indicating what would be changed
        nginx-proxy --check -d www.example.com -n wordpress
    
    Exit Codes::
    
            0            no changes were made or would have been made (check-mode)
            1            changes were made or would have been made (check-mode)
            2            fatal error prevented command completion
    
    Notes::
    
        # also supports the v13.0 syntax
        nginx-proxy www.example.com wordpress
    
        # template (preconfigured for 80, 443, 12320, 12321, 12322)
        /etc/nginx/sites-available/container.tmpl
    

Creating a container (wordpress, bridged)
-----------------------------------------

Continuing from the earlier inithooks example, we'll create a TurnKey
Wordpress container using the bridged network configuration.

1. Create the container::

    # lxc-create -n wp1 -f /etc/lxc/bridged.conf -t turnkey -- wordpress -i /root/inithooks.conf
    
    This could have been shortened because -i|--inithooks now defaults to /root/inithooks.conf
    # lxc-create -n wp1 -f /etc/lxc/bridged.conf -t turnkey -- wordpress
    
2. Start the container in the background::

    # lxc-start -d -n wp1

Creating a container (wordpress, NAT)
-------------------------------------

Now we'll create a second TurnKey Wordpress container. 
We'll also use the NAT bridge as it requires some
extra steps to expose the containers services to the network.

Additionally, we'll also specify an APT proxy (preconfigured on the 
TurnKey LXC appliance) so other containers can leverage the cache.

1. Create the container::

    # lxc-create -n wp2 -f /etc/lxc/natbridge.conf -t turnkey -- wordpress -i /root/inithooks.conf -x http://192.168.121.1:3142
    
    This could have been shortened because natbridge.conf is the default config and inithooks defaults to /root/inithooks.conf
    # lxc-create -n wp2 -t turnkey -- wordpress -x http://192.168.121.1:3142


2. Start the container in the background::

    # lxc-start -d -n wp2

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

4. Destroy the container::

    # lxc-destroy -n wp2

   or combine steps three and four::

    # lxc-destroy -f -n wp2

.. _inithooks: http://www.turnkeylinux.org/docs/inithooks

