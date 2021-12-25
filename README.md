# Ansible role for network configuration using systemd-networkd

This role translates YAML into a [systemd-networkd](https://www.freedesktop.org/software/systemd/man/systemd-networkd.html)
configuration.
"systemd-networkd is a system service that manages networks".
For DNS resolution, you can choose between systemd-resolved and resolvconf.

Warning:
This role **deletes** `/etc/network/interfaces` and disables the legacy
`networking.service`.

## Requirements

An apt based system that has systemd-networkd preinstalled,
like [Ubuntu](https://www.ubuntu.com/) or [Debian](https://www.debian.org/).

## Quick Start

First, you should be familiar with how to configure systemd-networkd,
so that you know what you aim for.
Useful links:

* https://wiki.archlinux.org/index.php/Systemd-networkd
* https://www.freedesktop.org/software/systemd/man/networkd.conf.html
* https://www.freedesktop.org/software/systemd/man/systemd.netdev.html
* https://www.freedesktop.org/software/systemd/man/systemd.network.html

Then, a playbook using this role looks like the following.

```yml
- hosts: router01
  become: true
  roles:
    - role: systemd-networkd
      systemd_network_confs:
        # /etc/systemd/networkd.conf.d/*.conf files are configured here; see below for detailed description
      systemd_network_netdevs:
        # /etc/systemd/network/*.netdev files are configured here; see below for examples
      systemd_network_networks:
        # /etc/systemd/network/*.network files are configured here; see below for examples
```

Hereby, the layout of the `systemd_network_netdevs` and `systemd_network_networks`
role variables can be easiest understood looking at the following examples.
In simple configurations with no virtual network interfaces you only need the
`systemd_network_networks` role variable.

### Example: Wired adapter using DHCP

```yml
systemd_network_networks:
  enp1s0:
    Network:
      DHCP: ipv4

##### Turns into:
#
##### /etc/systemd/network/enp1s0.network
#
# [Match]
# Name=enp1s0
#
# [Network]
# DHCP=ipv4
```

### Example: Wired adapter using a static IP

```yml
systemd_network_networks:
  enp1s0:
    Network:
      Address: 10.1.10.9/24
      Gateway: 10.1.10.1
      DNS: 8.8.8.8

##### Turns into:
#
##### /etc/systemd/network/enp1s0.network
#
# [Match]
# Name=enp1s0
#
# [Network]
# Address=10.1.10.9/24
# Gateway=10.1.10.1
# DNS=8.8.8.8
```

### Example: Bridge with two ports, using a static IP

```yml
systemd_network_netdevs:
  my_bridge:
    NetDev:
      Kind: bridge

systemd_network_networks:
  enp1s0 enp2s0:
    Network:
      Bridge: my_bridge

  my_bridge:
    Network:
      Address: 10.1.10.9/24
      Gateway: 10.1.10.1
      DNS: 8.8.8.8

##### Turns into:
#
##### /etc/systemd/network/my_bridge.netdev
#
# [NetDev]
# Name=my_bridge
# Kind=bridge
#
##### "/etc/systemd/network/enp1s0 enp2s0.network"
#
# [Match]
# Name=enp1s0 enp2s0
#
# [Network]
# Bridge=my_bridge
#
##### /etc/systemd/network/my_bridge.network
#
# [Match]
# Name=my_bridge
#
# [Network]
# Address=10.1.10.9/24
# Gateway=10.1.10.1
# DNS=8.8.8.8
```

## Detailed Description

* A key `x` in `systemd_network_confs` causes an INI file
  `/etc/systemd/networkd.conf.d/x.conf` to be created.
  Under `systemd.network_confs['x']`,
  the first-layer keys turn into INI sections,
  the second-layer keys turn into INI properties and
  the values thereunder turn into the respective INI values.

* A key `x` in `systemd_network_netdevs` causes an INI file
  `/etc/systemd/network/x.netdev` to be created.
  By default it contains:

  ```ini
  [NetDev]
  Name=x
  ```

  `systemd_network_netdevs['x']` analogously turns into INI.

* A key `x` in `systemd_network_networks` causes an INI file
  `/etc/systemd/network/x.network` to be created.
  By default it contains:

  ```ini
  [Match]
  Name=x
  ```

  `systemd_network_networks['x']` analogously turns into INI, with one
  special behaviour:
  If you specify a `Match` key (turning into a `[Match]` section) inside
  `systemd_network_networks['x']`, then the default `Name=x` **does not apply**.
  This makes it possible to
  [match on different things than name](https://www.freedesktop.org/software/systemd/man/systemd.network.html#%5BMatch%5D%20Section%20Options).

If you need multiple sections (respectively properties) with the same
name, then specify that section (respectively property) once and give
a list at the next YAML layer.
Examples:

```yml
# Example for multiple [Route] sections
systemd_network_networks:
  enp1s0:
    Network:
      Address: 192.168.1.2/24
    Route:
      - Destination: 192.168.5.0/24
        Gateway: 192.168.1.1
      - Destination: 192.168.6.0/24
        Gateway: 192.168.1.1

##### Turns into:
#
##### /etc/systemd/network/enp1s0.network
#
# [Match]
# Name=enp1s0
#
# [Network]
# Address=192.168.1.2/24
#
# [Route]
# Destination=192.168.5.0/24
# Gateway=192.168.1.1
#
# [Route]
# Destination=192.168.6.0/24
# Gateway=192.168.1.1
```

```yml
# Example for multiple VLAN= properties
systemd_network_networks:
  enp1s0:
    Network:
      VLAN:
        - enp1s0.10
        - enp1s0.11

##### Turns into:
#
##### /etc/systemd/network/enp1s0.network
#
# [Match]
# Name=enp1s0
#
# [Network]
# VLAN=enp1s0.10
# VLAN=enp1s0.11
```

## DNS resolver

This role automatically disables resolvconf, enables systemd-resolved
and configures the `/etc/resolv.conf` symlink correctly for systemd-resolved.
Using systemd-resolved (as this role does by default) is recommended because
systemd-resolved reads the DNS servers from your systemd network configuration,
whereas the DNS servers for resolvconf can't be configured using this role.

If you want to use resolvconf instead, there's a role variable for that:

```yml
systemd_network_dns_resolver: resolvconf
```

## Upload extra files

There also is a role variable `systemd_network_copy_files` which
simply takes a list of which each element is passed to the Ansible
[copy module](https://docs.ansible.com/ansible/latest/modules/copy_module.html).

### WireGuard client example

`systemd_network_copy_files` can for example be used for WireGuard key
files, as illustrated in the following example.
Installing WireGuard is out of the scope of this role and can be done using
[our wireguard role](https://github.com/stuvusIT/wireguard).

```yml
systemd_network_copy_files:
  - content: INSERT-CLIENT-PRIVATE-KEY-HERE
    dest: /etc/wireguard/private-key
    group: systemd-network
    mode: 0640

systemd_network_netdevs:
  wg0:
    NetDev:
      Kind: wireguard
      Description: WireGuard
    WireGuard:
      # NOTE: PrivateKeyFile needs at least systemd version 242.
      PrivateKeyFile: /etc/wireguard/private-key
      ListenPort: 51820
    WireGuardPeer:
      # Use a list, so we can add further peers if we desire.
      - PublicKey: INSERT-SERVER-PUBLIC-KEY-HERE
        AllowedIPs: 0.0.0.0/0
        Endpoint: 1.2.3.4:51820
        PersistentKeepalive: 20

systemd_network_networks:
  enp1s0:
    Network:
      DHCP: ipv4
  wg0:
    Network:
      Address: 192.168.30.11/24
```

## Role variable defaults

| Name                                        | Default              | Description                                                                                                                                          |
| :------------------------------------------ | :------------------- | :--------------------------------------------------------------------------------------------------------------------------------------------------- |
| `systemd_network_netdevs`                   | `{}`                 | [#detailed-description](#detailed-description)                                                                                                       |
| `systemd_network_networks`                  | `{}`                 | [#detailed-description](#detailed-description)                                                                                                       |
| `systemd_network_copy_files`                | `[]`                 | [#upload-extra-files](#upload-extra-files)                                                                                                           |
| `systemd_network_dns_resolver`              | `"systemd-resolved"` | [#dns-resolver](#dns-resolver)                                                                                                                       |
| `systemd_network_keep_existing_definitions` | `false`              | If false, the role deletes existing `.netdev` and `.network` files in `/etc/systemd/network/` and also deletes the corresponding network interfaces. |
