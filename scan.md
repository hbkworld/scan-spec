# Network Discovery and Configuration Protocol for Embedded Devices, Version 1.0
## Hottinger Baldwin Messtechnik GmbH

## Scope

This document describes the protocol for the HBM network discovery and
configuration for embedded devices.

## License

This document ist published under the terms of the [Creative Commons No
Derivative Works (CC BY-ND)](https://creativecommons.org/licenses/by-nd/4.0/).

## Overview

The discovery mechanism is intended to enable client programs (typically on a
PC) to communicate to embedded devices (devices throughout this
document) regardless of their current IP configuration. To achieve this,
devices need to perform the following actions:

-   Send [announce notifications](#announce-notification) on a regurlar basis regardless of the
    current IP settings,

-   Process incoming [network configuration
    requests](#configuration-request).

## Requirements

### Device

Devices must be capable to communicate using the IP protocol. Moreover, they
must also be capable to send and receive IPv4 multicast datagrams.  Because the
information sent is encoded in [JSON](http://www.ietf.org/rfc/rfc4627.txt), a
[JSON](http://www.ietf.org/rfc/rfc4627.txt) parser is highly recommended.

### Client

Clients must be capable to communicate using the IP protocol. Moreover, they
must also be capable to send and receive IP multicast datagrams.  Because
information sent is encoded in [JSON](http://www.ietf.org/rfc/rfc4627.txt), a
[JSON](http://www.ietf.org/rfc/rfc4627.txt) parser must be used.

## Network Communication

Communication is always carried out using IPv4 multicasting. The multicast
address (mutlicast group) to be used is `239.255.77.76` and `239.255.77.77`.
This restricts communication to the Local Scope of the Administratively Scoped
IP Multicast (please see RFC2365 for further information). The multicast
address allows discovery of devices beyond router boundaries, if routers are
configured to route multicast packets. The destination IP port for
announcements of devices is `31416`. Request and responses for configuration
go to port `31417`.

Please note that some operating systems, notably Linux, can be
configured to perform [reverse path filtering](#reverse-path-filtering).
Reverse path filtering is a mechanism that IP packets, which do not
originate from a source address that matches the net of the receiving
interface are not delivered to the process that makes a
recvfrom/recvmsg. The Linux kernel defaults to not perform Reverse Path
filtering, but Linux distributions might enable this feature. For
embedded devices Reverse Path filtering must be switched off, otherwise
the multicast communication will not work as expected. Windows XP and
Windows 7 do not perform Reverse Path filtering. 

## Datagram Description

Because the maximum size of a multicast datagram is 65535 bytes, the
size of request and response datagrams must never exceed 65535 bytes.
However, the MTU of Ethernet is 1500 bytes, so sending datagrams of
65535 bytes requires IP fragmentation. Smaller IP stacks may not be
capable of IP fragmentation, so the size of the request and response
datagrams should not exceed 1500 bytes.

The data datagrams are encoded using [JSON-RPC 2.0](http://www.jsonrpc.org/specification).

The [JSON-RPC](http://www.jsonrpc.org/specification) datagrams sent
model a request-response or notification communication. There are
dedicated requests (requests addressing exactly one device) and
non-dedicated notifications (notification addressing all devices that
might be reachable). The dedicated requests feature an "id" key to match
requests and responses.

### Device Announcement

The device announces itself periodically on multicast group
`239.255.77.76`, port `31416`. The client must receive and filter
multicast telegrams to gather information about the devices.

Keep in mind: It is possible to receive announcement from the same
device via several interfaces and via a router. Hence you might receive
different announcements carrying the same uuid.

#### Announcement Notification Datagram {#announce-notification}

~~~~ {.javascript}
{
  "jsonrpc": "2.0",
  "method": "announce",
  "params": {
    "apiVersion": <string>,
    "device": {
      "uuid": <string>,
      "name": <string>,
      "type": <string>,
      "label": <string>,
      "familyType": <string>,
      "firmwareVersion": <string>,
      "isRouter": true|false
    },
    "netSettings": {
      "interface": {
        "name": <string>,
        "type": <string>,
        "description": <string>,
        "configurationMethod": <string>,
        "ipv4": [
          {
            "address": <string>,
            "netmask": <string>
          }
        ],
        "ipv6": [
          {
            "address": <string>,
            "prefix": <unsigned int>
          }
        ]
      }, 
    },
    -- the following section is optional
    "router": {
      "uuid": <string>
    },
    -- the following section is optional
    "services": [
      { "type": <string>, "port": <number> },
      { "type": <string>, "port": <number> }
    ],
    "expiration": <unsigned int>
  }
}
~~~~

##### Explanation of Keys

  -------------------------------------------------------------------------
  key                                  description
  ------------------------------------ ------------------------------------
  "apiVersion"                            Version of the HBM discovery protocol

  -------------------------------------------------------------------------

###### Device Subtree

The device subtree describes some properties of the device.

  -------------------------------------------------------------------------
  key                                  description
  ------------------------------------ ------------------------------------
  "uuid"                               A string containing the worldwide unique 
                                       ID of the device. That might be the MAC
                                       address of a network interface card.
                                       This uuid is necessary to address
                                       devices by a dedicated request.

  "name"                               An optional string containing the
                                       name of the device.

  "type"                               A string describing the type of the
                                       device, e.g. for a QuantumX MX840
                                       this will contain "MX840".

  "label"                              An optional string with the text as printed on the device type label.

  "familyType"                         A string describing the family type
                                       of the device, e.g. QuantumX or
                                       PMX.

  "firmwareVersion"                    A string containing the firmware
                                       version of the device.

  "isRouter"                           this key is send with value true if the module acts as a IP router.

  -------------------------------------------------------------------------

  : Device Subtree Keys

###### Interface Subtree

The interface subtree describes the properties of an network interface.

  -------------------------------------------------------------------------
  key                                  description
  ------------------------------------ ------------------------------------
  "name"                               A string containing the name of the
                                       interface. For embedded Linux systems
                                       typically something like eth0, eth1,
                                       ...

  "type"                               An optional string containing the
                                       type of the interface. For QuantumX
                                       systems it might be useful to
                                       distinguish Ethernet and Firewire
                                       interfaces.

  "description"                        An optional string containing
                                       some additional information.
                                       QuantumX devices report whether
                                       the interface is on the front or
                                       back side.

  "configurationMethod"                A string enumeration describing how
                                       the network settings configured on
                                       the device during the startup.
                                       Currently the values *manual*,
                                       *dhcp* and *routerSolicitation* are
                                       valid. *This key is now deprecated
									   and shall not be used anymore.*

  "ipv4"                               An array containig all IPv4
                                       addresses of the interface with
                                       their netmask

  "ipv6"                               An array containig all IPv6
                                       addresses of the interface with
                                       their prefix
  -------------------------------------------------------------------------

  : Interface Subtree Keys

###### Router Subtree

The optional router subtree describes if a device is connected to a
special router device, for instance the CX27 in QuantumX.

  -------------------------------------------------------------------------
  key                                  description
  ------------------------------------ ------------------------------------
  "uuid"                               A string containing the unique ID of
                                       the router the device is connected
                                       to.
  -------------------------------------------------------------------------

  : Router Subtree Keys

###### Service Subtree

The optional service subtree might be used to deliver the IP port under
which the client can reach different services on the device. So devices might
e.g. specify how to connect to the data acquisition service.

The content of the service subtree is totally device specific and not
specified in this document.

-------------------------------------------------------------------------
  key                                  description
------------------------------------ ------------------------------------
  "type"                               Name of the service

  "port"                               IP port of the service
-------------------------------------------------------------------------
  : Service Subtree Keys

###### Expiration Key

The announcement is repeated periodically. Expiration specifies a time
period in seconds starting from the last received announcement.  Another
announcement has to arrive before the expiration time has elapsed.
Otherwise the device is considered to be not available anymore.  Reasons
can be a device power down/restart, a network problem or a device
failure.

### Network Configuration

Network configuration is only specified and allowed for
configuring IPv4 addresses for devices. IPv6 configuration via the
mechanism described below is not necessary because with IPv6 devices
will always have a properly configured link local IPv6 address.

Network configuration requests are to be send using the multicast group
`239.255.77.77`, port `31417`. The client will always receive a
JSON-PRC response from the same multicast group and the same port indicating whether the request is accepted.
On success it will contain a result member with result=0. On failure it contains an error object.

Succesfull requests are processed after the response was send. 

If necessary, network interfaces will be restarted automatically when
their configuration is changed.

After the interface was started again, announcements will be send periodically containing the new interface state.

It might also be necessary to reboot the device in order to apply the
new interface configuration. The reboot is performed automatically when
necessary. In this case the response will return with result=4.
The device will continue announcing itself, when back online
[“anncounce” notification](#announce-notification).

#### Request Datagram {#configuration-request}

~~~~ {.javascript}
{
  "jsonrpc": "2.0",
  "method": "configure",
  "params": {
    "device": {
      "uuid": <string>
    },
    "netSettings": {
      "interface": {
        "name": <string>,
        "ipv4": {
          "manualAddress": <string>,
          "manualNetmask": <string>
        }
        "configurationMethod": <string>
      },
    },
    "ttl": <number>
  },
  "id": <string>
}
~~~~

Network configuration request datagrams are dedicated requests, so only
the device addressed by request/configure/device/uuid must answer with a
network configuration response datagram.

Because network configuration request datagrams are dedicated requests,
they contain an "id" key to match requests and responses. Please look
into the [JSON-RPC 2.0
specification](http://www.jsonrpc.org/specification) for an explanation
of the keys "method", "params" and "id".

Request and response are send via multicast. Hence everyone that joined
the multicast group is going to receive both messages.  Please make sure
that "id" has a unique value. Otherwise you might confuse responses.

##### Explanation of Keys

###### Params Subtree Keys

Contains all required parameters for the request.

  -------------------------------------------------------------------------
  key                                  description
  ------------------------------------ ------------------------------------
  "ttl"                                An optional key which limits the
                                       number of router hops a configure
                                       request/response can cross. Leaving
                                       out this key should default to a ttl
                                       (Time to live) of 1 when sending
                                       datagrams, so no router boundary is
                                       crossed.
  -------------------------------------------------------------------------

  : Params Subtree Keys

####### Device Subtree

Conatains the device to be configured.

  -------------------------------------------------------------------------
  key                                  description
  ------------------------------------ ------------------------------------
  "uuid"                               This string contains the unique ID
                                       of the device that should be
                                       configured. The uuid itself must be
                                       gathered from an announce datagram.
  -------------------------------------------------------------------------

  : Device Subtree Keys

####### Interface Subtree

Contains the information how the interface identified by "name" shall be
configured.

  -------------------------------------------------------------------------
  key                                  description
  ------------------------------------ ------------------------------------
  "name"                               A string containing the interface
                                       name that should be configured. The
                                       interface name must be gathered from
                                       an announce datagram.

  "configurationMethod"                A string enumeration describing how
                                       the network settings configured on
                                       the device during the startup.
                                       Currently the values *manual* and
                                       *dhcp* are valid.

  "manualAddress"                      An optional string containing the
                                       manual IP address that should be
                                       configured on the device.

  "manualNetmask"                      An optional string containing the
                                       manual IP netmask that should be
                                       configured on the device.
  -------------------------------------------------------------------------

  : Interface Subtree Keys

#### Response Datagram

A [JSON-RPC](http://www.jsonrpc.org/specification) response will be returned to
the sender of the request. It indicates whether the request was successful or
not (See the specification of [JSON-RPC](http://www.jsonrpc.org/specification)
for explanation of error reporting).

# Reverse Path Filtering {#reverse-path-filtering}
Several Linux Distributions enable reverse path filtering by
default. This prevents the HBM discovery mechanism from working. You can
disable reverse path filtering by editing /etc/sysctl.conf and setting
the following entries:

    net.ipv4.conf.all.rp_filter = 0
    net.ipv4.conf.default.rp_filter = 0

After editing you have to reboot your machine.


## Glossary

IP: Internet Protocol

JSON: JavaScript Object Notation
<http://www.ietf.org/rfc/rfc4627.txt>

JSON-RPC: JavaScript Object Notation Remote Procedure Call
<http://www.jsonrpc.org/specification>

MTU: Maximum Transfer Unit
The maximum amount of bytes that can be transferred without
fragmentation.
