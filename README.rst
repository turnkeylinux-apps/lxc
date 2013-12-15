LXC - Linux Containers
======================

`LXC`_ can be thought of as the middle ground between a chroot on steriods and
a full fledged virtual machine. LXC is a userspace interface to the Linux
kernel containment features (such as namespaces and control groups), allowing
sandboxing processes from one another and controlling their resource
allocations. 

This appliance includes all the standard features in `TurnKey Core`_, and on
top of that:

- Includes custom `TurnKey LXC template`_:

    - Download and create a container of any TurnKey appliance.
    - Insert specified inithooks.conf into container for preseeding.
    - Supports configuration of network link (e.g., br0, natbr0, none).
    - Generic enough to be used on any LXC enabled distribution.
    
- LXC appliance configurations:

    - Preconfigured network bridge interface (br0).
    - Preconfigured network NAT bridge interface (natbr0).
    - Preconfigured dnsmasq on natbr0 providing DHCP and DNS services.
      Containers can be referenced by hostname or hostname.local.lxc
    - Includes convenience script (`iptables-nat`_) for exposing a NAT'ed
      containers port to the network.
    - Includes example inithooks configuration for preseeding.
    - IP forwarding and control groups enabled.

See the `Usage documentation`_ for further details.

Credentials *(passwords set at first boot)*
-------------------------------------------

-  Webmin, SSH: username **root**

.. _LXC: http://linuxcontainers.org
.. _TurnKey Core: http://www.turnkeylinux.org/core
.. _TurnKey LXC template: https://github.com/turnkeylinux-apps/lxc/blob/master/overlay/usr/share/lxc/templates/lxc-turnkey
.. _iptables-nat: https://github.com/turnkeylinux-apps/lxc/blob/master/overlay/usr/local/bin/iptables-nat
.. _Usage documentation: https://github.com/turnkeylinux-apps/lxc/tree/master/docs

