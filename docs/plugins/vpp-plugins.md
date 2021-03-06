# VPP plugins

---

## GoVPPMux plugin

The `govppmux` is a core Agent plugin which **allows other plugins to access the VPP** independently at each other by means of connection multiplexing.

Any plugin (core or external) that interacts with the VPP can ask `govppmux` to get its own, potentially customized communication channel to access running VPP instance. Behind the scene, **all channels share the same connection** created during the plugin initialization using `govpp` core.

### Connection

Tho GoVPP multiplexer supports two connection types:
  - using socket client
  - using shared memory
    
By default the GoVPP connects uses the **socket client**. This option can be changed using environment variable `GOVPPMUX_NOSOCK` which fallback to the **shared memory** - in that case, the plugin connects to that instance of the VPP which uses the default shared memory segment prefix.
 
The **default behaviour assumes that there is only a single VPP running** in a sand-boxed environment together with the agent (e.g. through containerization). In case the VPP runs with customized SHM prefix, or **there are several VPP instances** running side-by-side, the **GoVPP needs to know the prefix in order to connect to the desired VPP 
instance**. The prefix has to be put into the GoVPPMux configuration file (called `govpp.conf`) with the key `shm-prefix` and value matching the VPP shared memory prefix name.

### Multiplexing

The `NewAPIChannel` call returns a new API channel for communication with VPP via the `GoVPP` core. It uses default buffer sizes for the request and reply go channels (by default both are 100 messages long).

If it is expected that the VPP may get overloaded at peak loads (for example if the user plugin sends configuration requests in bulks) then it is recommended to use `NewAPIChannelBuffered` and increase the buffer size for requests appropriately. Similarly, `NewAPIChannelBuffered` allows to configure the size of the buffer for responses. This is also useful since the buffer for responses is also used to carry VPP notifications and statistics which may temporarily rapidly grow in size and frequency. By increasing the reply channel size, the probability of dropping messages from VPP decreases at the cost of increased memory footprint.

### Trace
 
Duration of the VPP binary api call can be measured using trace feature. These data are logged after every event (any resynchronization, new interfaces, bridge domains, fib entries etc.). Enable trace in the `govpp.conf`: 
 
`trace-enabled: true` or  `trace-enabled: false`
  
The trace feature is disabled by default (if there is no config available). 

**Configuration example**

The plugin allows to configure parameters of the VPP health-check probe. Possible items to configure:

  - `health check probe interval`: time between health check probes
  - `health check reply timeout`: if the reply does not arrive until timeout elapses probe is considered failed
  - `health check threshold`: number of consequent failed health checks until an error is reported
  - `reply timeout`: if the reply from channel does not arrive until timeout elapses, the request fails
  - `shm-prefix`: used for connection to a VPP instance which is not using default shared memory prefix
  - `resync-after-reconnect`: allows to run resync after the VPP reconnection
  
## Interface plugin

The VPP interface plugin is a base plugin able to setup various types of **VPP interfaces** and also manages **the 
interface status** (state data, optionally published back to the database) and **interface and DHCP lease 
notifications**. The VPP provides multiple interface types, following are supported in the VPP-Agent:

  - Sub-interface
  - Software loopback
  - DPDK (physical) interface
  - Memory interface (memif)
  - Tap (version 1 and 2)
  - Af-packet interface
  - VxLAN tunnel
  - IPSec tunnel
  - VmxNet3 interface
  - Bond interfaces

Other types are specified as undefined for the vpp-agent purpose.
  
All the interfaces except DPDK (physical) can be created or removed directly in the VPP if all necessary conditions are 
met (for example an Af-packet interface requires a VEth as a host in order to attach to it. The **DPDK interfaces** 
(PCI, physical) **can be only configured**, cannot be added or removed.
  
The VPP uses multiple commands and/or binary APIs to create and configure shared boundaries. The vpp-agent is 
simplifying this processes, providing the single model with specific extensions depending on required interface type. 
The northbound configuration is translated to a sequence of binary API calls using GoVPP library. All interface types 
except the DPDK (physical) interface can be directly created or removed in the VPP. Physical interfaces can be only 
configured. The Af-packet interface is dependent on host-interface of the VEth type, and cannot exist without it.

The interface plugin defines following [model][interface-model]. The model is divided into two parts - **the first, 
which contains fields common for all interface types, and the second, exclusive for given interface type**. Note that 
there are even exceptions in the common part, where not all fields can be defined on every interface type. For example, 
a physical address cannot be set to Af-packet or to DPDK interface despite the field exists in the definition. If this 
is done, the value is silently ignored.

Mandatory fields are **interface type** and **logical name**. The name is limited to 63 characters. The interface may 
or may not define unique fields for the given type.

All interfaces use the same key for every type. It means the name has to be incomparable among all interfaces. See the
[key reference][key-reference] to learn more about the interface keys.

The **interface logical name is for the vpp-agent use only**. It serves as a reference for other models (bridge domains 
for instance). The VPP works with indexes, which are generated when the new interface instance is created. The index is 
a unique integer for identification and future VPP internal references. In order to parse name and index, the VPP-Agent
stores a tag to the VPP via binary API - it's a name-to-index entry which helps the VPP-Agent to classify interface 
name and assign the correct index to it. 

### Unnumbered interfaces

Every **interface can have assigned one or more IPv4 or IPv6 addresses** or their combination. Besides this, the VPP 
interface can be set as unnumbered; it **takes the IP address of another interface and performs as it has the same IP 
address** as the target instance. 

To set an interface as unnumbered, omit `ip_addresses` and set `interface_with_ip` in the model. The target interface 
has to exist (if not, the unnumbered interface will be postponed and configured later when the required interface 
appears) and must contain at least one IP address. 

### Maximum transmission unit

The MTU is the size of the largest protocol data unit that can be communicated in a single network layer transaction. 
The VPP interfaces allow setting custom MTU size. For this purpose, there is a field called `mtu` available in the 
interface model. If the field is left empty, the VPP uses a default value of the MTU.

The vpp-agent offers an option to automatically set the MTU of specific size to every created interface via config file 
([example][interface-config-file]). If the global MTU value is set to non-zero value, but interface definition contains 
its own different value, the **local value is preferred before the global** one.

!!! note
    MTU is not available for VxLan tunnels and IPSec tunnel interfaces (despite they contain incriminated fields).

### Interface types

Type-specific interface fields in the VPP-Agent are defined in special structures called links. Type of the interface's 
link must match the type of the interface itself. Description of main differences for given interface types:

**Bond interface** was created to support Link Aggregation Control Protocol (LACP)). Network bonding is a process of 
combing or joining two or more network interfaces together into a single interface. The bond interface in addition 
supports various modes and load balancing techniques.

**Sub-interface** is an interface type derived from other interfaces adding VLAN ID to it. The sub-interface link of type `SubInterface` requires a name of the parent (super) interface, and ID of the VLAN.

**Software loopback** interface type does not have any special fields.

**DPDK interface** or the physical interface cannot be created or removed directly by the VPP-Agent. The PCI interface 
has to be connected to the VPP - they should be configured for use by the Linux kernel and shut down. VPP-Agent can set 
the interface IP, MAC or set it as enabled. No special configuration fields are defined.

**Memory interface** or shared memory packet interface ( or simply memif) provides high-performance packet transmit and 
reception between VPPs. The memory interface defines additional fields in `MemifLink`. The most important is a socket 
filename - the VPP by default contains default socket filename with zero index which can be used to create memif 
interfaces (the vpp-agent needs to be resynced to register it). 

**Tap interface** exists in two versions but only the latter can be configured. TAPv2 is based on virtio (it knows that 
runs in a virtual environment, and cooperates with the hypervisor). The `TapLink` provides several setup options 
available for the TAPv2. 

!!! danger "important"
    The first version of the Tap interface is no longer supported by the VPP, thus cannot be managed by the vpp-agent. 

Field `version` sets the desired version (should be set only to 2), `host_if_name` allows to set a custom name for the 
TAP interface in the host OS (optional, if not set it will be autogenerated). The option `to_microservice` puts the 
host TAP to the desired namespace right after creation. Please refer the [interface model][interface-model] to know 
which fields are defined for specific TAP interface version.

**Af-packet interface** is the VPP "host" interface, its main purpose is to connect it to the host via the OS VEth 
interface. Because of this, the Af-packet cannot exist without its host interface. If the desired VEth is missing, the 
configuration is postponed. If the host interface was removed, vpp-agent un-configures and caches related Af-packets. 
The `AfpacketLink` contains only one field with the name of the host interface (as defined in the 
[Linux interface plugin][linux-interface-plugin-guide].

**VxLan tunnel** endpoint (VTEP) defines `VxlanLink` with source and destination address, a VXLan network identifier 
(VNI) and a name of the optional multicast interface.

**IPSec virtual tunnel** interface (VTI) provides a routable interface type for terminating IPSec tunnels and an easy 
way to define protection between sites to form an overlay network. The IPSec tunnel defines `IPSecLink` with various 
options like local and remote IP address, local and remote security parameter index (SPI) or cryptographic algorithms 
for encryption and authentication. Available cryptographic algorithms are determined from types supported by the VPP.

**VmxNet3 interface** is a virtual network adapter designed to deliver high performance in virtual environments. 
VmxNet3 requires VMware tools to be used. Type-specific fields are defined in `VmxNet3Link` structure.

**Configuration example**

**1. Using key-value database** put the proto-modelled data with the correct key for interfaces 
([key reference][key-reference]).

Example value of common interface configuration, since loopback type does not use any type specific fields.
```json
{  
    "name":"loop1",
    "type":"SOFTWARE_LOOPBACK",
    "enabled":true,
    "phys_address":"7C:4E:E7:8A:63:68",
    "ip_addresses":[  
        "192.168.25.3/24",
        "172.125.45.1/24"
    ],
    "mtu":1478
}
```

Use `etcdctl` to put compacted loop interface key-value entry:
```bash
etcdctl put /vnf-agent/vpp1/config/vpp/v2/interfaces/loop1 '{"name":"loop1","type":"SOFTWARE_LOOPBACK","enabled":true,"phys_address":"7C:4E:E7:8A:63:68","ip_addresses":["192.168.25.3/24","172.125.45.1/24"],"mtu":1478}'
```

Another example with memif type interface. Note memif-specific fields in `memif` section:
```json
{  
    "name":"memif1",
    "type":"MEMIF",
    "enabled":true,
    "phys_address":"4E:93:2A:38:A7:77",
    "ip_addresses":[  
        "172.125.40.1/24"
    ],
    "mtu":1478,
    "memif":{  
        "master":true,
        "id":1,
        "socket_filename":"/tmp/memif1.sock",
        "secret":"secret"
    }
}
```

Use `etcdctl` to put compacted memif interface key-value entry:
```bash
etcdctl put /vnf-agent/vpp1/config/vpp/v2/interfaces/memif1 '{"name":"memif1","type":"MEMIF","enabled":true,"phys_address":"4E:93:2A:38:A7:77","ip_addresses":["172.125.40.1/24"],"mtu":1478,"memif":{"master":true,"id":1,"socket_filename":"/tmp/memif1.sock","secret":"secret"}}'
```

**2. Using REST:**

The REST currently supports only retrieval of the existing configuration. The following command can be used to read all interfaces via cURL:

```bash
curl -X GET http://localhost:9191/dump/vpp/v2/interfaces
```

The interface type-specific calls:
```bash
curl -X GET http://localhost:9191/dump/vpp/v2/interfaces/loopback
curl -X GET http://localhost:9191/dump/vpp/v2/interfaces/ethernet
curl -X GET http://localhost:9191/dump/vpp/v2/interfaces/memif
curl -X GET http://localhost:9191/dump/vpp/v2/interfaces/tap
curl -X GET http://localhost:9191/dump/vpp/v2/interfaces/vxlan
curl -X GET http://localhost:9191/dump/vpp/v2/interfaces/afpacket
```

**3. Using GRPC:**

Prepare the interface data:
```go
import (
	"github.com/ligato/vpp-agent/api/models/vpp"
	"github.com/ligato/vpp-agent/api/models/vpp/interfaces"
)

memoryInterface := &vpp.Interface{
        Name:        "memif1",
        Type:        interfaces.Interface_MEMIF,
        Enabled:     true,
        IpAddresses: []string{"192.168.10.1/24"},
        Link: &interfaces.Interface_Memif{
            Memif: &interfaces.MemifLink{
                Id:             1,
                Master:         true,
                SocketFilename: "/tmp/memif1.sock",
            },
        },
    }
```

Prepare the GRPC config data:
```go
import (
	"github.com/ligato/vpp-agent/api/configurator"
	"github.com/ligato/vpp-agent/api/models/vpp"
)

config := &configurator.Config{
		VppConfig: &vpp.ConfigData{
			Interfaces: []*interfaces.Interface{
				memoryInterface,
			},
			
		}
	}
```

The config data can be combined with any other VPP or Linux configuration.

Update the data via the GRPC using the client of the `ConfigurationClient` type (read more about how to prepare the GRPC connection and about other CRUD methods in the [GRPC tutorial][grpc-tutorial]):
```go
import (
	"github.com/ligato/vpp-agent/api/configurator"
)

response, err := client.Update(context.Background(), &configurator.UpdateRequest{Update: config, FullResync: true})
```

### Interface status

The interface status is a **collection of interface data and identification** (tagged name, internal name, index) which can be stored to an external database or send a notification.

The vpp-agent waits on several types of VPP events, like creation, deletion or counter update. The event is processed and all the data extracted. Using the collected information, VPP-Agent builds a notification which is sent to all registered publishers (databases). All errors occurred are also stored. The statistics readout is performed every second.

!!! danger "Important"
    The interface status is disabled by default, since no default publishers are defined. Use interface plugin config file to define it (see the [example config file][interface-config-file])   

The interface state [model][interface-state-model] is represented by two objects: `InterfaceState` with all the status data, and `InterfaceNotification` which is a wrapper around the state with additional information required to send status as a notification.

Keys used for interface status and errors:

```
// Interface status
/vnf-agent/<label>/vpp/status/v2/interface/<name>

// Errors
/vnf-agent/<label>/vpp/status/v2/interface/errors/<name>
```

The interface status is meant to be read-only, published only by the VPP-Agent.

Data provided by interface status:

  * identification (logical and internal name, software interface index, interface type)
  * administrative and operational status
  * physical address, MTU value
  * used duplex 
  * statistics (received, transmitted or dropped packets, bytes, error count, etc)

**Interface status usage**

To read status data, use any tool with access to a given database (**etcdctl** for the ETCD, **redis-cli** for Redis, **boltbrowser** for BoltDB, etc). 

Let's read interface status data with `etcdctl`:
```
etcdctl get --prefix /vnf-agent/vpp1/vpp/status/v2/interface/
```

The command above returns status for all interfaces, which have it currently stored inside the ETCD. To read the status just for single interface, add its name to the end:
```
etcdctl get --prefix /vnf-agent/vpp1/vpp/status/v2/interface/loop1
etcdctl get --prefix /vnf-agent/vpp1/vpp/status/v2/interface/memif1
```

Do not forget to enable status in the config file, otherwise, the status will not be published to the database.

## L2 plugin

The VPP L2 plugin is a base plugin which can be used to configure link-layer related configuration items to the VPP, notably **bridge domains**, **forwarding tables** (FIBs) and **VPP cross connects**. The L2 plugin is strongly dependent on [interface plugin][interface-plugin-guide] since every configuration item has some kind of dependency on it. 

### Bridge domains

The L2 bridge domain is a set of interfaces which share the same flooding or broadcasting characteristics. Every bridge domain contains several attributes (MAC learning, unicast forwarding, flooding, ARP termination table) which can be enabled or disabled. The bridge domain is identified by unique ID, which is managed by the plugin and cannot be defined 
from outside.   

The L2 plugin defines individual proto definition for every configuration item, so the bridge domain is also defined by its own [model][bd-model]. The definition consists of common bridge domain fields (forwarding, learning...), list of assigned interfaces and ARP termination table. Bridge domain interfaces are only referenced - the configuration of the interface itself (IP address, ...) lays on the interface plugin. Every referenced interface also contains bridge domain-specific fields (BVI and split horizon group). Note that **only one BVI interface is allowed per bridge domain**.

The bridge domain does not have any mandatory fields except the logical name. In that case, an empty bridge domain with no interface or ARP entries is created. The bridge domain logical name is for the plugin-use and also as a reference in other configuration types dependent on the specific bridge domain. The bridge domain index can be obtained via 
retrieve, but the value is only informative since it cannot be changed, set or referenced.

**Configuration example**

**1. Using the key-value database** put the proto-modelled data with the correct key for bridge domains to the database ([key reference][key-reference]).

Example value:

```json
{
    "name": "bd1",
    "learn": true,
    "flood": true,
    "forward": true,
    "unknown_unicast_flood": true,
    "arp_termination": true,
    "interfaces": [
        {
            "name": "if1",
            "split_horizon_group": 0,
            "bridged_virtual_interface": true
        },
        {
            "name": "if2",
            "split_horizon_group": 0
        },
        {
            "name": "if2",
            "split_horizon_group": 0
        }
    ],
    "arp_termination_table": [
        {
            "ip_address": "192.168.10.10",
            "phys_address": "a7:65:f1:b5:dc:f6"
        },
        {
            "ip_address": "10.10.0.1",
            "phys_address": "59:6C:45:59:8E:BC"
        }
    ] 
}
```

Use `etcdctl` to put compacted key-value entry:

```bash
etcdctl put /vnf-agent/vpp1/config/vpp/l2/v2/bridge-domain/bd1 '{"name":"bd1","learn":true,"flood":true,"forward":true,"unknown_unicast_flood":true,"arp_termination":true,"interfaces":[{"name":"if1","split_horizon_group":0,"bridged_virtual_interface":true},{"name":"if2","split_horizon_group":0},{"name":"if2","split_horizon_group":0}],"arp_termination_table":[{"ip_address":"192.168.10.10","phys_address":"a7:65:f1:b5:dc:f6"},{"ip_address":"10.10.0.1","phys_address":"59:6C:45:59:8E:BC"}]}'
```

To remove the configuration:

```bash
etcdctl del /vnf-agent/vpp1/config/vpp/l2/v2/bridge-domain/bd1
```

**2. Using REST:**

The REST currently supports only retrieving of the existing configuration. The following command can be used to read all bridge domains via cURL:

```bash
curl -X GET http://localhost:9191/dump/vpp/v2/bd
```

**3. Using GRPC:**

Prepare the bridge domain data:
```go
import (
	"github.com/ligato/vpp-agent/api/models/vpp"
	"github.com/ligato/vpp-agent/api/models/vpp/l2"
)

bd := &l2.BridgeDomains_BridgeDomain{
		Name:                "bd1",
		Flood:               false,
		UnknownUnicastFlood: false,
		Forward:             true,
		Learn:               true,
		ArpTermination:      false,
		Interfaces: []*l2.BridgeDomains_BridgeDomain_Interfaces{
			{
				Name: "if1",
				BridgedVirtualInterface: true,
			}
		},
	}
```

Prepare the GRPC config data:
```go
import (
	"github.com/ligato/vpp-agent/api/configurator"
	"github.com/ligato/vpp-agent/api/models/vpp"
)

config := &configurator.Config{
		VppConfig: &vpp.ConfigData{
			BridgeDomains: []*l2.BridgeDomain{
				bd,
			},
			
		}
	}
```

The config data can be combined with any other VPP or Linux configuration.

Update the data via the GRPC using the client of the `ConfigurationClient` type (read more about how to prepare the GRPC connection and about other CRUD methods in the [GRPC tutorial][grpc-tutorial]):
```go
import (
	"github.com/ligato/vpp-agent/api/configurator"
)

response, err := client.Update(context.Background(), &configurator.UpdateRequest{Update: config, FullResync: true})
```

### Forwarding tables  

An L2 forwarding information base (FIB) (also known as a forwarding table) can be used in network bridging or routing to **find the output interface to which the incoming packet should be forwarded**. FIB entries are created also by the VPP itself (like for interfaces within the bridge domain with enabled MAC learning). The VPP-Agent allows to configuring static FIB entries for a combination of interface and bridge domain.

A FIB entry is defined in the following [model][fib-model]. To successfully configure FIB entry, a MAC address, interface and bridge domain must be provided. Also, the condition that the interface is a part of the bridge domain has to be fulfilled. The interface and the bridge domain are referenced with logical names. Note that the FIB entry will 
not appear in the VPP until all conditions are met.  

**Configuration example**

**1. Using the key-value database** put the proto-modelled data with the correct key for FIB to the database ([key reference][key-reference]).

Example value:
```json
{
    "bridge_domain": "bd1",
    "outgoing_interface": "tap1",
    "phys_address": "62:89:C6:A3:6D:5C",
    "action":"FORWARD",
    "static_config": true,
    "bridged_virtual_interface": false
}
```

Use `etcdctl` to put compacted key-value entry:
```bash
etcdctl put /vnf-agent/vpp1/config/vpp/l2/v2/fib/bd1/mac/62:89:C6:A3:6D:5C '{"phys_address":"62:89:C6:A3:6D:5C","bridge_domain":"bd1","outgoing_interface":"tap1","action":"FORWARD","static_config":true,"bridged_virtual_interface":false}'
```

**2. Using REST:**

The REST currently supports only retrieving of the existing configuration. The following command can be used to read all FIBs via cURL:
```bash
curl -X GET http://localhost:9191/dump/vpp/v2/fib
```

**3. Using GRPC:**

Prepare the FIB data:
```go
import (
	"github.com/ligato/vpp-agent/api/models/vpp"
	"github.com/ligato/vpp-agent/api/models/vpp/l2"
)

fib := &l2.FIBEntry{
		PhysAddress: "a7:35:45:55:65:75",
		BridgeDomain: "bd1",
		Action: l2.FIBEntry_FORWARD,
		OutgoingInterface: "if1",
	}
```

Prepare the GRPC config data:
```go
import (
	"github.com/ligato/vpp-agent/api/configurator"
	"github.com/ligato/vpp-agent/api/models/vpp"
)

config := &configurator.Config{
		VppConfig: &vpp.ConfigData{
			Fibs: []*l2.FIBEntry {
				bd,
			},	
		}
	}
```

The config data can be combined with any other VPP or Linux configuration.

Update the data via the GRPC using the client of the `ConfigurationClient` type (read more about how to prepare the GRPC connection and about other CRUD methods in the [GRPC tutorial][grpc-tutorial]):
```go
import (
	"github.com/ligato/vpp-agent/api/configurator"
)

response, err := client.Update(context.Background(), &configurator.UpdateRequest{Update: config, FullResync: true})
```

### Cross connects

The Agent supports L2 cross connect feature, which sets interface pair to cross connect mode. All packets received on the first interface are transmitted to the second interface. This mode is not bidirectional by default, both interfaces have to be set.

The cross-connect mode uses a very simple [model][xc-model] which defines references to transmit and receive interface. Both interfaces are mandatory fields and must exist in order to set the mode successfully. If one or both of them are missing, the configuration is postponed. 

**Configuration example**

**1. Using the key-value database** put the proto-modelled data with the correct key for cross connect to the database ([key reference][key-reference]).

Example value:
```json
{
    "receive_interface": "if1",
    "transmit_interface": "if2"
}
```

Use `etcdctl` to put compacted key-value entry:
```bash
etcdctl put /vnf-agent/vpp1/config/vpp/l2/v2/xconnect/if1 '{"receive_interface":"if1","transmit_interface":"if2"}'
```
**2. Using REST:**

The REST currently supports only retrieving of the existing configuration. The following command can be used to read all cross-connect entries via cURL:

```bash
curl -X GET http://localhost:9191/dump/vpp/v2/xc
```

**3. Using GRPC:**

Prepare the cross-connect data:
```go
import (
	"github.com/ligato/vpp-agent/api/models/vpp"
	"github.com/ligato/vpp-agent/api/models/vpp/l2"
)

xc := &l2.XConnectPair{
		ReceiveInterface: "if1",
		TransmitInterface: "if2",
	}
```

Prepare the GRPC config data:
```go
import (
	"github.com/ligato/vpp-agent/api/configurator"
	"github.com/ligato/vpp-agent/api/models/vpp"
)

config := &configurator.Config{
		VppConfig: &vpp.ConfigData{
			XconnectPairs: []*l2.XConnectPair {
				xc,
			},	
		}
	}
```

The config data can be combined with any other VPP or Linux configuration.

Update the data via the GRPC using the client of the `ConfigurationClient` type (read more about how to prepare the GRPC connection and about other CRUD methods in the [GRPC tutorial][grpc-tutorial]):
```go
import (
	"github.com/ligato/vpp-agent/api/configurator"
)

response, err := client.Update(context.Background(), &configurator.UpdateRequest{Update: config, FullResync: true})
```

## L3 plugin

The VPP L3 plugin is capable of configuring **ARP** entries (including **proxy ARP**), **VPP routes** and **IP neighbor** feature. The L3 plugin is dependent on the [interface plugin][interface-plugin-guide] in many aspects since several configuration items require the interface to be already present.

### ARP entries  

The L3 plugin defines descriptor managing the VPP implementation of the address resolution protocol. **ARP is a communication protocol for discovering the MAC address associated with a given IP address**. In terms of the plugin, the ARP entry is uniquely identified by the interface and IP address.

The northbound plugin API defines ARP in the following [model][arp-model]. Every ARP entry is defined with MAC address, IP address and an Interface. The two latter are a part of the key.

**Configuration example**

**1. Using the key-value database** put the proto-modelled data with the correct key for ARP to the database ([key reference][key-reference]).

Example value:
```json
{  
    "interface":"if1",
    "ip_address":"192.168.10.21",
    "phys_address":"59:6C:45:59:8E:BD",
    "static":true
}
```

Use `etcdctl` to put compacted key-value entry:
```bash
etcdctl put /vnf-agent/vpp1/config/vpp/v2/arp/tap1/192.168.10.21 '{"interface":"tap1","ip_address":"192.168.10.21","phys_address":"59:6C:45:59:8E:BD","static":true}'
```

To remove configuration:

```bash
etcdctl del /vnf-agent/vpp1/config/vpp/v2/arp/tap1/192.168.10.21
```

**2. Using REST:**

The REST currently supports only retrieving of the existing configuration. The following command can be used to read all ARP entries via cURL:

```bash
curl -X GET http://localhost:9191/dump/vpp/v2/arps
```

**3. Using GRPC:**

Prepare the ARP data:
```go
import (
	"github.com/ligato/vpp-agent/api/models/vpp"
	"github.com/ligato/vpp-agent/api/models/vpp/l3"
)

arp := &l3.ARPEntry{
		Interface: "if1",
		IpAddress: "10.20.30.40/24",
		PhysAddress: "55:65:75:a7:35:45",
	}
```

Prepare the GRPC config data:
```go
import (
	"github.com/ligato/vpp-agent/api/configurator"
	"github.com/ligato/vpp-agent/api/models/vpp"
)

config := &configurator.Config{
		VppConfig: &vpp.ConfigData{
			Arps: []*l3.ARPEntry {    
			    arp,
			},	
		}
	}
```

The config data can be combined with any other VPP or Linux configuration.

Update the data via the GRPC using the client of the `ConfigurationClient` type (read more about how to prepare the GRPC connection and about other CRUD methods in the [GRPC tutorial][grpc-tutorial])):
```go
import (
	"github.com/ligato/vpp-agent/api/configurator"
)

response, err := client.Update(context.Background(), &configurator.UpdateRequest{Update: config, FullResync: true})
```

### Proxy ARP

Support for the VPP proxy ARP - a technique by which a proxy device an a given network answers the ARP queries for an IP address from a different network. Proxy ARP IP address ranges are defined via the respective binary API call. Besides this, desired interfaces have to be enabled for the proxy ARP feature.    

The Proxy ARP [model][l3-model] is located in the common L3 model proto file. The model defines a list of IP address ranges and another list of interfaces. Naturally, the interface has to exist in order to be set as enabled for ARP 
proxy. 

**Configuration example**  

**1. Using the key-value database** put the proto-modelled data with the correct key for proxy ARP to the database ([key reference][key-reference]).

Example value:
```json
{  
    "interfaces":[  
        {  
            "name":"tap1"
        },
        {  
            "name":"tap2"
        }
    ],
    "ranges":[  
        {  
            "first_ip_addr":"10.0.0.1",
            "last_ip_addr":"10.0.0.3"
        }
    ]
}
```

Use `etcdctl` to put compacted key-value entry:
```bash
etcdctl put /vnf-agent/vpp1/config/vpp/v2/proxyarp-global/settings '{"interfaces":[{"name":"tap1"},{"name":"tap2"}],"ranges":[{"first_ip_addr":"10.0.0.1","last_ip_addr":"10.0.0.3"}]}'
```

**2. Using REST:**

The REST currently supports only retrieving of the existing configuration. The Proxy ARP uses two cURL commands to obtain interfaces and ranges: 

```bash
curl -X GET http://localhost:9191/dump/vpp/v2/proxyarp/interfaces
curl -X GET http://localhost:9191/dump/vpp/v2/proxyarp/ranges

```

**3. Using GRPC:**

Prepare the Proxy-ARP data:
```go
import (
	"github.com/ligato/vpp-agent/api/models/vpp"
	"github.com/ligato/vpp-agent/api/models/vpp/l3"
)

xc := &l2.XConnectPair{
		ReceiveInterface: "if1",
		TransmitInterface: "if2",
	}
```

Prepare the GRPC config data:
```go
import (
	"github.com/ligato/vpp-agent/api/configurator"
	"github.com/ligato/vpp-agent/api/models/vpp"
)

proxyArp := &l3.ProxyARP{
		Interfaces: []*l3.ProxyARP_Interface{
			{
				Name: "if1",
			},
		},
		Ranges: []*l3.ProxyARP_Range{
			{
				FirstIpAddr: "10.0.0.1",
				LastIpAddr: "10.0.0.255",
			},
		},
	}
```

The config data can be combined with any other VPP or Linux configuration.

Update the data via the GRPC using the client of the `ConfigurationClient` type (read more about how to prepare the GRPC connection and about other CRUD methods in the [GRPC tutorial][grpc-tutorial]):
```go
import (
	"github.com/ligato/vpp-agent/api/configurator"
)

response, err := client.Update(context.Background(), &configurator.UpdateRequest{Update: config, FullResync: true})
```

### Routes 

The VPP routing table lists routes to particular network destinations. Routes are grouped in virtual routing and forwarding (VRF) tables, with default table of index 0. A route can be assigned to different VRF table directly in the modeled data.

The VPP route [model][route-model] defines all the base route fields (destination IP, next hop, interface) and other specifications like weight or preference. The model also distinguishes amidst several route types:

  **1. Intra-VRF route:** defines a route where the forwarding is done only in the specific VRF or according to the outgoing interface.
  **2. Inter-VRF route:** forwarding is being done by a lookup into different VRF routes. This kind of route does not  expect outgoing interface being defined. 
  **3. Drop route:** such a route drops network communication designed for IP address specified.
  
**Configuration example**

**1. Using the key-value database** put the proto-modelled data with the correct key for routes to the database 
([key reference][key-reference]).

Example data (intra-VRF route):
```json
{  
    "type": "INTRA_VRF",
    "dst_network":"10.1.1.3/32",
    "next_hop_addr":"192.168.1.13",
    "outgoing_interface":"tap1",
    "weight":6
}
```

Use `etcdctl` to put compacted intra-VRF route:
```bash
etcdctl put /vnf-agent/vpp1/config/vpp/v2/route/vrf/0/dst/10.1.1.3/32/gw/192.168.1.13 '{"type":"INTRA_VRF","dst_network":"10.1.1.3/32","next_hop_addr":"192.168.1.13","outgoing_interface":"tap1","weight":6}'
```

Another example (inter-VRF route)
```json
{  
    "type":"INTER_VRF",
    "dst_network":"1.2.3.4/32",
    "via_vrf_id":1
}
```

Use `etcdctl` to put compacted inter-VRF route:
```bash
etcdctl put /vnf-agent/vpp1/config/vpp/v2/route/vrf/0/dst/1.2.3.4/32/gw '{"type":"INTER_VRF","dst_network":"1.2.3.4/32","via_vrf_id":1}'
```

**2. Using REST:**

The REST currently supports only retrieving of the existing configuration. The following command can be used to read all routes via cURL:

```bash
curl -X GET http://localhost:9191/dump/vpp/v2/routes
```

**3. Using GRPC:**

Prepare the VPP route data:
```go
import (
	"github.com/ligato/vpp-agent/api/models/vpp"
	"github.com/ligato/vpp-agent/api/models/vpp/l3"
)

route := &l3.Route{
		VrfId:             0,
		DstNetwork:        "10.1.1.3/32",
		NextHopAddr:       "192.168.1.13",
		Weight:            60,
		OutgoingInterface: "if1",
	}
```

Prepare the GRPC config data:
```go
import (
	"github.com/ligato/vpp-agent/api/configurator"
	"github.com/ligato/vpp-agent/api/models/vpp"
)

config := &configurator.Config{
		VppConfig: &vpp.ConfigData{
			Routes: []*l3.Route {}
				route,
			},	
		}
	}
```

The config data can be combined with any other VPP or Linux configuration.

Update the data via the GRPC using the client of the `ConfigurationClient` type (read more about how to prepare the GRPC connection and about other CRUD methods in the [GRPC tutorial][grpc-tutorial]):
```go
import (
	"github.com/ligato/vpp-agent/api/configurator"
)

response, err := client.Update(context.Background(), &configurator.UpdateRequest{Update: config, FullResync: true})
```
 
### IP scan-neighbor

The VPP IP scan-neighbor feature is used to enable or disable periodic IP neighbor scan. The [model][l3-model] allows to set various scan intervals, timers or delays and can be set to IPv4 mode, IPv6 mode or both of them.

**Configuration example**  

**1. Using the key-value database** put the proto-modelled data with the correct key for IP scan neighbor to the database ([key reference][key-reference]).

Example data:
```json
{  
    "mode":"BOTH",
    "scan_interval":11,
    "max_proc_time":36,
    "max_update":5,
    "scan_int_delay":16,
    "stale_threshold":26
}
```

Use `etcdctl` to put compacted data:
```bash
etcdctl put /vnf-agent/vpp1/config/vpp/v2/ipscanneigh-global/settings '{"mode":"BOTH","scan_interval":11,"max_proc_time":36,"max_update":5,"scan_int_delay":16,"stale_threshold":26}'
```

**2. Using REST:**

IP scan neighbor configuration type does not support REST.

**3. Using GRPC:**

Prepare the IP scan neighbor data:
```go
import (
	"github.com/ligato/vpp-agent/api/models/vpp"
	"github.com/ligato/vpp-agent/api/models/vpp/l3"
)

ipScanNeigh := &l3.IPScanNeighbor{
		Mode:           l3.IPScanNeighbor_BOTH,
		ScanInterval:   11,
		MaxProcTime:    36,
		MaxUpdate:      5,
		ScanIntDelay:   16,
		StaleThreshold: 26,
	}
```

Prepare the GRPC config data:
```go
import (
	"github.com/ligato/vpp-agent/api/configurator"
	"github.com/ligato/vpp-agent/api/models/vpp"
)

config := &configurator.Config{
		VppConfig: &vpp.ConfigData{
			IpscanNeighbor: *l3.IPScanNeighbor {  
				ipScanNeigh,
			},	
		}
	}
```

The config data can be combined with any other VPP or Linux configuration.

Update the data via the GRPC using the client of the `ConfigurationClient` type (read more about how to prepare the GRPC connection and about other CRUD methods in the [GRPC tutorial][grpc-tutorial]):
```go
import (
	"github.com/ligato/vpp-agent/api/configurator"
)

response, err := client.Update(context.Background(), &configurator.UpdateRequest{Update: config, FullResync: true})
```

## IPSec plugin

The IPSec plugin allows to configure **security policy databases** and **security associations** to the VPP, and also handles relations between the SPD and SA or between SPD and an assigned interface. Note that the IPSec tunnel interfaces are not a part of IPSec plugin (their configuration is handled in the [VPP interface plugin][interface-plugin-guide]).

### Security policy database

Security policy database (SPD) specifies policies that determine the disposition of all the inbound or outbound IP traffic from either the host or the security gateway. The SPD is bound to an SPD interface and contains a list of policy entries (security table). Every policy entry points to VPP security association. Security policy database is defined in the common IPSec [model][ipsec-model]. 

The SPD defines its own unique index within VPP. The user has an option to set its own index (it is not generated in the VPP, as for example for access lists) in `uint32` range. The index is a mandatory field in the model because it serves as a unique identifier also in the VPP-Agent and is a part of the SPD key as well. The attention has to be paid 
defining an index in the model. Despite the field is a `string` format, it only accepts plain numbers. Attempt to set any non-numeric characters ends up with an error.     

Every policy entry has field `sa_index`. This is the security association reference in the security policy database. The field is mandatory, missing value causes an error during configuration.

The SPD defines two bindings: the security association (for every entry) and the interface. The interface binding is important since VPP needs to create an empty SPD first, which cannot be done without it. All the policy entries are configured after that, where the SA is expected to exist. Since every security policy database entry is configured independently, VPP-Agent can configure IPSec SPD only partially (depending on available binding).

All the binding can be resolved by the VPP-Agent. 

**Configuration example**

**1. Using the key-value database** put the proto-modelled data with the correct key for security policy databases to the database ([key reference][key-reference]).

Example value:
```json
{
  "index":"1",
  "interfaces": [
    { "name": "tap1" }
  ],
  "policy_entries": [
    {
      "priority": 10,
      "is_outbound": false,
      "remote_addr_start": "10.0.0.1",
      "remote_addr_stop": "10.0.0.1",
      "local_addr_start": "10.0.0.2",
      "local_addr_stop": "10.0.0.2",
      "action": 3,
      "sa_index": "20"
    },
    {
      "priority": 10,
      "is_outbound": true,
      "remote_addr_start": "10.0.0.1",
      "remote_addr_stop": "10.0.0.1",
      "local_addr_start": "10.0.0.2",
      "local_addr_stop": "10.0.0.2",
      "action": 3,
      "sa_index": "10"
    }
  ]
}
```

Use `etcdctl` to put compacted key-value entry:
```bash
etcdctl put /vnf-agent/vpp1/config/vpp/ipsec/v2/spd/1 '{"index":"1","interfaces":[{"name":"tap1"}],"policy_entries":[{"priority":10,"is_outbound":false,"remote_addr_start":"10.0.0.1","remote_addr_stop":"10.0.0.1","local_addr_start":"10.0.0.2","local_addr_stop":"10.0.0.2","action":3,"sa_index":"20"},{"priority":10,"is_outbound":true,"remote_addr_start":"10.0.0.1","remote_addr_stop":"10.0.0.1","local_addr_start":"10.0.0.2","local_addr_stop":"10.0.0.2","action":3,"sa_index":"10"}]}'
```

**2. Using REST:**

IPSec currently does not support REST.

**3. Using GRPC:**

Prepare the IPSec SPD data:
```go
import (
	"github.com/ligato/vpp-agent/api/models/vpp"
	"github.com/ligato/vpp-agent/api/models/vpp/ipsec"
)

spd := ipsec.SecurityPolicyDatabase{
		Index: "1",
		Interfaces: []*ipsec.SecurityPolicyDatabase_Interface{
			{
				Name: "if1",
			}
		},
		PolicyEntries: []*ipsec.SecurityPolicyDatabase_PolicyEntry{
			{
				Priority:        10,
				IsOutbound:      false,
				RemoteAddrStart: "10.0.0.1",
				RemoteAddrStop:  "10.0.0.1",
				LocalAddrStart:  "10.0.0.2",
				LocalAddrStop:   "10.0.0.2",
				Action:          3,
				SaIndex:         "1",
			},
		},
	}
```

Prepare the GRPC config data:
```go
import (
	"github.com/ligato/vpp-agent/api/configurator"
	"github.com/ligato/vpp-agent/api/models/vpp"
)

config := &configurator.Config{
		VppConfig: &vpp.ConfigData{
			IpsecSpds: []*ipsec.SecurityPolicyDatabase {
				spd,
			},	
		}
	}
```

The config data can be combined with any other VPP or Linux configuration.

Update the data via the GRPC using the client of the `ConfigurationClient` type (read more about how to prepare the GRPC connection and about other CRUD methods in the [GRPC tutorial][grpc-tutorial]):
```go
import (
	"github.com/ligato/vpp-agent/api/configurator"
)

response, err := client.Update(context.Background(), &configurator.UpdateRequest{Update: config, FullResync: true})
```

### Security associations

The VPP security association (SA) is a set of IPSec specifications negotiated between devices establishing and IPSec relationship. The SA includes preferences for the authentication type, IPSec protocol or encryption used when the IPSec connection is established.

Security association is defined in the common IPSec [model][ipsec-model]. 

The SA uses the same indexing system as SPD. The index is a user-defined unique identifier of the security association in `uint32` range. Similarly to SPD, the SA index is also defined as `string` type field in the model but can be set only to numerical values. Attempt to set other values causes processing errors. The SA has no dependencies on other 
configuration types.

**Configuration example**

**1. Using the key-value database** put the proto-modelled data with the correct key for security associations to the database ([key reference][key-reference]).

Example value:
```json
{
  "index":"1",
  "spi": 1001,
  "protocol": 1,
  "crypto_alg": 1,
  "crypto_key": "4a506a794f574265564551694d653768",
  "integ_alg": 2,
  "integ_key": "4339314b55523947594d6d3547666b45764e6a58"
}
```

Use `etcdctl` to put compacted key-value entry:
```bash
etcdctl put /vnf-agent/vpp1/config/vpp/ipsec/v2/sa/1 '{"index":"1","spi":1001,"protocol":1,"crypto_alg":1,"crypto_key":"4a506a794f574265564551694d653768","integ_alg":2,"integ_key":"4339314b55523947594d6d3547666b45764e6a58"}'
```

**2. Using REST:**

IPSec currently does not support REST.

**3. Using GRPC:**

Prepare the IPSec SA data:
```go
import (
	"github.com/ligato/vpp-agent/api/models/vpp"
	"github.com/ligato/vpp-agent/api/models/vpp/ipsec"
)

sa := ipsec.SecurityAssociation{
		Index:          "1",
		Spi:            1001,
		Protocol:       1,
		CryptoAlg:      1,
		CryptoKey:      "4a506a794f574265564551694d653768",
		IntegAlg:       2,
		IntegKey:       "4339314b55523947594d6d3547666b45764e6a58",
		EnableUdpEncap: true,
	}
```

Prepare the GRPC config data:
```go
import (
	"github.com/ligato/vpp-agent/api/configurator"
	"github.com/ligato/vpp-agent/api/models/vpp"
)

config := &configurator.Config{
		VppConfig: &vpp.ConfigData{
			IpsecSas: []*ipsec.SecurityAssociation {
				sa,
			},	
		}
	}
```

The config data can be combined with any other VPP or Linux configuration.

Update the data via the GRPC using the client of the `ConfigurationClient` type (read more about how to prepare the GRPC connection and about other CRUD methods in the [GRPC tutorial][grpc-tutorial]):
```go
import (
	"github.com/ligato/vpp-agent/api/configurator"
)

response, err := client.Update(context.Background(), &configurator.UpdateRequest{Update: config, FullResync: true})
```

## Punt plugin

The punt plugin provides several options for how to configure the VPP to allow a specific IP traffic to be punted to the host TCP/IP stack. The plugin supports **punt to the host** (either directly, or **via Unix domain socket**) and registration of **IP punt redirect** rules.

### Punt to host stack

All the incoming traffic matching one of the VPP interface addresses, and also matching defined L3 protocol, L4 protocol, and port - and would be otherwise dropped - will be instead punted to the host. If a Unix domain socket path is defined (optional), traffic will be punted via socket. All the fields which serve as a traffic filter are mandatory.

The punt plugin defines the following [model][punt-model] which grants support for two main configuration items defined by different northbound keys. L3/L4 protocol in the key is defined as a `string` value, however, the value is transformed to numeric representation in the VPP binary API. The usage of L3 protocol `ALL` is exclusive for IP punt to 
host (without socket registration) in the VPP API. If used for the IP punt with socket registration, the vpp-agent calls the binary API twice with the same parameters for both, IPv4 and IPv6.

!!! danger "Important"
    In order to configure a punt to host via Unix domain socket, a specific VPP startup-config is required. The attempt to set punt without it results in errors in VPP. Example tartup-config `punt { socket /tmp/socket/punt }`. The path has to match with the one in the northbound data. 

**Configuration example**

**1. Using the key-value database** put the proto-modelled data with the correct key for punt to host to the database ([key reference][key-reference]). 

Example value:
```json
{  
    "l3_protocol":"IPv4",
    "l4_protocol":"UDP",
    "port":9000,
    "socket_path":"/tmp/socket/path"
}
```

Use `etcdctl` to put compacted key-value entry:
```bash
etcdctl put /vnf-agent/vpp1/config/vpp/v2/tohost/l3/IPv4/l4/UDP/port/9000 {"l3_protocol":"IPv4","l4_protocol":"UDP","port":9000,"socket_path":"/tmp/socket/path"}
```

**2. Using REST:** 

Punt currently does not support REST.

**3. Using GRPC:**

Prepare the Punt data:
```go
import (
	"github.com/ligato/vpp-agent/api/models/vpp"
	"github.com/ligato/vpp-agent/api/models/vpp/punt"
)

punt := &punt.ToHost{
    L3Protocol: punt.L3Protocol_IPv4,
    L4Protocol: punt.L4Protocol_UDP,
    Port:       9000,
    SocketPath: "/tmp/socket/path",
}
```

Prepare the GRPC config data:
```go
import (
	"github.com/ligato/vpp-agent/api/configurator"
	"github.com/ligato/vpp-agent/api/models/vpp"
)

config := &configurator.Config{
    VppConfig: &vpp.ConfigData{
        PuntTohosts: []*punt.ToHost {
            punt,
        },	
    }
}
```

The config data can be combined with any other VPP or Linux configuration.

Update the data via the GRPC using the client of the `ConfigurationClient` type (read more about how to prepare the GRPC connection and about other CRUD methods in the [GRPC tutorial][grpc-tutorial]):
```go
import (
	"github.com/ligato/vpp-agent/api/configurator"
)

response, err := client.Update(context.Background(), &configurator.UpdateRequest{Update: config, FullResync: true})
```

### IP redirect

Defined as the IP punt, IP redirect allows a traffic matching given IP protocol to be punted to the defined TX interface and next hop IP address. All those fields have to be defined in the northbound proto-modeled data. Optionally, the RX interface can be also defined as an input filter.  

IP redirect is defined as `IpRedirect` object in the generated proto model. L3 protocol is defined as `string` value (transformed to numeric in VPP API call). The table is the same as before.

If L3 protocol is set to `ALL`, the respective API is called for IPv4 and IPv6 separately.

**Configuration example**

**1. Using the key-value database** put the proto-modelled data with the correct key for the IP redirect to the database ([key reference][key-reference]). 

Example value:
```json
{  
    "l3_protocol":"IPv4",
    "tx_interface":"tap1",
    "next_hop":"192.168.0.1"
}
```

Use `etcdctl` to put compacted key-value entry:
```bash
etcdctl put /vnf-agent/vpp1/config/vpp/v2/ipredirect/l3/IPv4/tx/tap1 '{"l3_protocol":"IPv4","tx_interface":"tap1","next_hop":"192.168.0.1"}'
```

**2. Using REST:** 

Punt currently does not support REST.

**3. Using GRPC:**

Prepare the IP redirect data:
```go
import (
	"github.com/ligato/vpp-agent/api/models/vpp"
	"github.com/ligato/vpp-agent/api/models/vpp/punt"
)

punt := &punt.IPRedirect{
    L3Protocol:  punt.L3Protocol_IPv4,
    TxInterface: "if1",
    NextHop:     "192.168.0.1",
}
```

Prepare the GRPC config data:
```go
import (
	"github.com/ligato/vpp-agent/api/configurator"
	"github.com/ligato/vpp-agent/api/models/vpp"
)

config := &configurator.Config{
    VppConfig: &vpp.ConfigData{
        PuntTohosts: []*punt.ToHost {
            punt,
        },	
    }
}
```

The config data can be combined with any other VPP or Linux configuration.

Update the data via the GRPC using the client of the `ConfigurationClient` type (read more about how to prepare the GRPC connection and about other CRUD methods in the [GRPC tutorial][grpc-tutorial]):
```go
import (
	"github.com/ligato/vpp-agent/api/configurator"
)

response, err := client.Update(context.Background(), &configurator.UpdateRequest{Update: config, FullResync: true})
```

### Limitations

Current limitations for a punt to host:
* The UDP configuration cannot be shown (or even configured) via the VPP cli.
* The VPP does not provide API to dump configuration, which takes the vpp-agent the opportunity to read existing entries and may case certain issues with resync.
* Although the vpp-agent supports the TCP protocol as the L4 protocol to filter incoming traffic, the current VPP version don't.
* Configured punt to host entry cannot be removed since the VPP does not support this option. The attempt to do so exits with an error.

Current limitations for a punt to host via unix domain socket:
* The configuration cannot be shown (or even configured) in the VPP cli.
* The vpp-agent cannot read registered entries since the VPP does not provide an API to do so.
* The VPP startup config punt section requires unix domain socket path defined. The VPP limitation is that only one path can be defined at the same time.

Current limitations for IP redirect:
* The VPP does not provide API calls to dump existing IP redirect entries. It may cause resync not to work properly.

### Known issues

* VPP issue: if the Unix domain socket path is defined in the startup config, the path has to exist, otherwise the VPP fails to start. The file itself can be created by the VPP.

## Access Control Lists plugin

Access control lists (ACLs) provide a means to filter packets by allowing a user to permit or deny specific IP traffic at defined interfaces. Access lists filter network traffic by controlling whether packets are forwarded or blocked at the router’s interfaces based on the criteria you specified within the access list.

The VPP-Agent acl plugin uses binary API of the VPP access control list plugin. The version of the VPP ACL plugin is displayed at the Agent startup (currently at 1.3). Every ACL consists from match (rules packet have to fulfill in order to be received) and action (what is done with the matched traffic). The current implementation rules for packet are 'ALLOW', 'DENY' and 'REFLECT'. The ACL is defined in the VPP-Agent northbound API [model][acl-model].

The Agent defines every access list with unique name. The VPP generates index, but the association is handled fully by the Agent. Every ACL must consist from single action and single type of match (IP match or MAC-IP match). The IP match (called IP rule) can be specified for variety of protocols, each with its own parameters. For example, the IP rule for IP protocol can define source and destination network the packet must match in order to execute defined action. Other supported protocols are TCP, UDP and ICMP. Single ACL rule can define multiple protocol-based rules. The MAC-IP match (MACIP rule) defines IP address + mask and MAC address + mask as filtering rules. Remember, that the IP rules and MACIP rules cannot be combined in the single ACL.

**Configuration example**

**1. Using the key-value database** put the proto-modelled data with the correct key for ACL entries to the database ([key reference][key-reference]).

Example configuration:
```json
{
    "name": "acl1",
    "interfaces": {
        "egress": [
            "tap1",
            "tap2"
        ],
        "ingress": [
            "tap3",
            "tap4"
        ]
    },
    "rules": [
        {
            "action": 1,
            "ip_rule": {
                "ip": {
                    "destination_network": "10.20.1.0/24",
                    "source_network": "192.168.1.2/32"
                },
                "tcp": {
                    "destination_port_range": {
                        "lower_port": 1150,
                        "upper_port": 1250
                    },
                    "source_port_range": {
                        "lower_port": 150,
                        "upper_port": 250
                    },
                    "tcp_flags_mask": 20,
                    "tcp_flags_value": 10
                }
            }
        }
    ]
}
```

Use `etcdctl` to put compacted key-value entry:
```
etcdctl put /vnf-agent/vpp1/config/vpp/acls/v2/acl/acl1 '{"name":"acl1","interfaces":{"egress":["tap1","tap2"],"ingress":["tap3","tap4"]},"rules":[{"action":1,"ip_rule":{"ip":{"destination_network":"10.20.1.0/24","source_network":"192.168.1.2/32"},"tcp":{"destination_port_range":{"lower_port":1150,"upper_port":1250},"source_port_range":{"lower_port":150,"upper_port":250},"tcp_flags_mask":20,"tcp_flags_value":10}}}]}'
```

**2. Using REST:**

Use following commands to obtain IP or MACIP based Access lists:
```bash
curl -X GET http://localhost:9191/dump/vpp/v2/acl/ip
curl -X GET http://localhost:9191/dump/vpp/v2/acl/macip

```

**3. Using GRPC:**

Prepare the ACL data:
```go
import acl "github.com/ligato/vpp-agent/api/models/vpp/acl"

acl := &acl.ACL{
		Name: "aclip1",
		Rules: []*acl.ACL_Rule{
			// ACL IP rule
			{
				Action: acl.ACL_Rule_PERMIT,
				IpRule: &acl.ACL_Rule_IpRule{
					Ip: &acl.ACL_Rule_IpRule_Ip{
						SourceNetwork:      "192.168.1.1/32",
						DestinationNetwork: "10.20.0.1/24",
					},
				},
			},
			// ACL ICMP rule
			{
				Action: acl.ACL_Rule_PERMIT,
				IpRule: &acl.ACL_Rule_IpRule{
					Icmp: &acl.ACL_Rule_IpRule_Icmp{
						Icmpv6: false,
						IcmpCodeRange: &acl.ACL_Rule_IpRule_Icmp_Range{
							First: 150,
							Last:  250,
						},
						IcmpTypeRange: &acl.ACL_Rule_IpRule_Icmp_Range{
							First: 1150,
							Last:  1250,
						},
					},
				},
			},
			// ACL TCP rule
			{
				Action: acl.ACL_Rule_PERMIT,
				IpRule: &acl.ACL_Rule_IpRule{
					Tcp: &acl.ACL_Rule_IpRule_Tcp{
						TcpFlagsMask:  20,
						TcpFlagsValue: 10,
						SourcePortRange: &acl.ACL_Rule_IpRule_PortRange{
							LowerPort: 150,
							UpperPort: 250,
						},
						DestinationPortRange: &acl.ACL_Rule_IpRule_PortRange{
							LowerPort: 1150,
							UpperPort: 1250,
						},
					},
				},
			},
			// ACL UDP rule
			{
				Action: acl.ACL_Rule_PERMIT,
				IpRule: &acl.ACL_Rule_IpRule{
					Udp: &acl.ACL_Rule_IpRule_Udp{
						SourcePortRange: &acl.ACL_Rule_IpRule_PortRange{
							LowerPort: 150,
							UpperPort: 250,
						},
						DestinationPortRange: &acl.ACL_Rule_IpRule_PortRange{
							LowerPort: 1150,
							UpperPort: 1250,
						},
					},
				},
			},
		},
		Interfaces: &acl.ACL_Interfaces{
			Ingress: []string{"tap1", "tap2"},
			Egress:  []string{"tap1", "tap2"},
		},
	}
```

Prepare the GRPC config data:
```go
import (
	"github.com/ligato/vpp-agent/api/configurator"
	"github.com/ligato/vpp-agent/api/models/vpp"
)

config := &configurator.Config{
		VppConfig: &vpp.ConfigData{
			Acls: []*acl.ACL {
				acl,
			},	
		}
	}
```

The config data can be combined with any other VPP or Linux configuration.

Update the data via the GRPC using the client of the `ConfigurationClient` type (read more about how to prepare the GRPC connection and about other CRUD methods in the [GRPC tutorial][grpc-tutorial]):
```go
import (
	"github.com/ligato/vpp-agent/api/configurator"
)

response, err := client.Update(context.Background(), &configurator.UpdateRequest{Update: config, FullResync: true})
```

## NAT plugin

Network address translation, or NAT is a method of remapping IP address space into another IP address space modifying address information in the packet header. The VPP-Agent Network address translation is control plane plugin for the VPP NAT implementation of NAT44. The NAT plugin is dependent on [interface plugin][interface-plugin-guide].

### NAT global config 

The global NAT configuration is a special case of data grouped under single key (it means no unique character is a part of the key, so there is always only one global NAT configuration). The data are used to enable NAT features (like forwarding), enable interfaces for NAT, define NAT IP addresses (address pools) or specify virtual reassembly. Interfaces marked to be enabled for NAT should be present in the VPP but if not, the Scheduler plugin caches the configuration for later use when the incriminated interface is available.

The global configuration is divided into several independent parts defining certain VPP NAT features:

  - **Forwarding** is a boolean field enabling or disabling forwarding.
  - **NAT interfaces** represent a list of interfaces which will be enabled for NAT. If the interface does not exist in   the VPP, it is cached and potentially configured later. Every interface is defined by its logical name and whether is   an inside or an outside interface. The output feature is also defined here.
  - **NAT interfaces** represent a list of interfaces which will be enabled for NAT. If the interface does not exist in   the VPP, it is cached and potentially configured later. Every interface is defined by its logical name and whether is   an inside or an outside interface. The output feature is also defined here.
  - **Address pools** is a list of IP addresses for given VRF table and with either enabled or disabled twice NAT.   Despite the name, only one address is defined in the single "pool" entry.
  - **Virtual reassembly** provides support for datagram fragmentation handling to allow correct recalculation of   higher-level checksums.

**Configuration example**

**1. Using the key-value database** put the proto-modelled data with the correct key for nat global config to the database ([key reference][key-reference]).

Example value:
```json
{  
    "nat_interfaces":[  
        {  
            "name":"if1"
        },
        {  
            "name":"if2",
            "output_feature":true
        },
        {  
            "name":"if3",
            "is_inside":true
        }
    ],
    "address_pool":[  
        {  
            "address":"192.168.0.1"
        },
        {  
            "address":"175.124.0.1"
        },
        {  
            "address":"10.10.0.1"
        }
    ],
    "virtual_reassembly":{  
        "timeout":10,
        "max_reassemblies":20,
        "max_fragments":10,
        "drop_fragments":true
    }
}
```

Use `etcdctl` to put compacted key-value entry:
```bash
etcdctl put /vnf-agent/vpp1/config/vpp/nat/v2/nat44-global/settings {"nat_interfaces":[{"name":"tap1"},{"name":"tap2","output_feature":true},{"name":"tap3","is_inside":true}],"address_pool":[{"address":"192.168.0.1"},{"address":"175.124.0.1"},{"address":"10.10.0.1"}],"virtual_reassembly":{"timeout":10,"max_reassemblies":20,"max_fragments":10,"drop_fragments":true}}
```

To remove the configuration:

```bash
etcdctl del /vnf-agent/vpp1/config/vpp/nat/v2/nat44-global/settings
```

**2. Using REST:**

The REST currently supports only retrieving of the existing configuration. The following command can be used to read the NAT global config via cURL:

```bash
curl -X GET http://localhost:9191/dump/vpp/v2/nat/global
```

**3. Using GRPC:**

Prepare the global NAT data:
```go
import (
	"github.com/ligato/vpp-agent/api/models/vpp"
	"github.com/ligato/vpp-agent/api/models/vpp/nat"
)

natGlobal := &nat.Nat44Global{
		Forwarding: false,
		NatInterfaces: []*nat.Nat44Global_Interface{
			{
				Name:          "if1",
				IsInside:      false,
				OutputFeature: false,
			},
			{
				Name:          "if2",
				IsInside:      false,
				OutputFeature: true,
			}
		},
		AddressPool: []*nat.Nat44Global_Address{
			{
				VrfId:    0,
				Address:  "192.168.0.1",
				TwiceNat: false,
			},
			{
				VrfId:    0,
				Address:  "175.124.0.1",
				TwiceNat: false,
			}
		},
		VirtualReassembly: &nat.VirtualReassembly{
			Timeout:         10,
			MaxReassemblies: 20,
			MaxFragments:    10,
			DropFragments:   true,
		},
	}
```

Prepare the GRPC config data:
```go
import (
	"github.com/ligato/vpp-agent/api/configurator"
	"github.com/ligato/vpp-agent/api/models/vpp"
)

config := &configurator.Config{
		VppConfig: &vpp.ConfigData{
			Nat44Global: &nat.Nat44Global {
				natGlobal,
			},
			
		}
	}
```

The config data can be combined with any other VPP or Linux configuration.

Update the data via the GRPC using the client of the `ConfigurationClient` type (read more about how to prepare the GRPC connection and about other CRUD methods in the [GRPC tutorial][grpc-tutorial]:
```go
import (
	"github.com/ligato/vpp-agent/api/configurator"
)

response, err := client.Update(context.Background(), &configurator.UpdateRequest{Update: config, FullResync: true})
```

### DNAT44

Destination network address translation (DNAT) allows transparently changing the destination IP address of an packet and performing the inverse function for any replies. Any router situated between two endpoints can perform this transformation of the packet. In the VPP-Agent, the DNAT configuration is a list of static and/or identity mappings 
labelled under single key.

The DNAT44 consists from two main parts - static mappings and identity mappings. The static mapping can be load balanced - if more than one local IP address is defined for single static mapping, the load balancer is automatically allowed for that mapping. 
THe DNAT44 contains a unique label serving as an identifier. However, the DNAT configuration is not limited, an arbitrary count of static and identity mappings can be listed under single label. 

**Configuration example**

**1. Using the key-value database** put the proto-modelled data with the correct key for nat44 to the database ([key reference][key-reference]).

Example value:
```json
{  
    "label":"dnat1",
    "st_mappings":[  
        {  
            "external_interface":"if1",
            "external_ip":"192.168.0.1",
            "external_port":8989,
            "local_ips":[  
                {  
                    "local_ip":"172.124.0.2",
                    "local_port":6500,
                    "probability":40
                },
                {  
                    "local_ip":"172.125.10.5",
                    "local_port":2300,
                    "probability":40
                }
            ],
            "protocol":"UDP",
            "twice_nat":"ENABLED"
        }
    ],
    "id_mappings":[  
        {  
            "ip_address":"10.10.0.1",
            "port":2525
        }
    ]
}
```

Use `etcdctl` to put compacted key-value entry:
```bash
etcdctl put /vnf-agent/vpp1/config/vpp/nat/v2/dnat44/dnat1 {"label":"dnat1","st_mappings":[{"external_interface":"tap1","external_ip":"192.168.0.1","external_port":8989,"local_ips":[{"local_ip":"172.124.0.2","local_port":6500,"probability":40},{"local_ip":"172.125.10.5","local_port":2300,"probability":40}],"protocol":"UDP","twice_nat":"ENABLED"}],"id_mappings":[{"ip_address":"10.10.0.1","port":2525}]}
```

To remove the configuration:

```bash
etcdctl del /vnf-agent/vpp1/config/vpp/nat/v2/dnat44/dnat1
```

**2. Using REST:**

The REST currently supports only retrieving of the existing configuration. The following command can be used to read the NAT global config via cURL:

```bash
curl -X GET http://localhost:9191/dump/vpp/v2/nat/dnat
```

**3. Using GRPC:**

Prepare the DNAT data:
```go
dNat := &nat.DNat44{
    Label: "dnat1",
    StMappings: []*nat.DNat44_StaticMapping{
        {
            ExternalInterface: "if1",
            ExternalIp:        "192.168.0.1",
            ExternalPort:      8989,
            LocalIps: []*nat.DNat44_StaticMapping_LocalIP{
                {
                    VrfId:       0,
                    LocalIp:     "172.124.0.2",
                    LocalPort:   6500,
                    Probability: 40,
                },
                {
                    VrfId:       0,
                    LocalIp:     "172.125.10.5",
                    LocalPort:   2300,
                    Probability: 40,
                },
            },
            Protocol: 1,
            TwiceNat: nat.DNat44_StaticMapping_ENABLED,
        },
    },
    IdMappings: []*nat.DNat44_IdentityMapping{
        {
            VrfId:     0,
            IpAddress: "10.10.0.1",
            Port:      2525,
            Protocol:  0,
        },
    },
}
```

Prepare the GRPC config data:
```go
import (
	"github.com/ligato/vpp-agent/api/configurator"
	"github.com/ligato/vpp-agent/api/models/vpp"
)

config := &configurator.Config{
    VppConfig: &vpp.ConfigData{
        Dnat44S: []*nat.DNat44 {
            natGlobal,
        },
        
    }
}
```

The config data can be combined with any other VPP or Linux configuration.

Update the data via the GRPC using the client of the `ConfigurationClient` type (read more about how to prepare the GRPC connection and about other CRUD methods in the [GRPC tutorial][grpc-tutorial]):
```go
import (
	"github.com/ligato/vpp-agent/api/configurator"
)

response, err := client.Update(context.Background(), &configurator.UpdateRequest{Update: config, FullResync: true})
```

## SR plugin

The `srplugin` is a Core Agent Plugin designed to configure Segment routing for IPv6 (SRv6) in the VPP.
Configuration managed by this plugin is modelled by [srv6 proto file][src6-model].

All configuration must be stored in ETCD using the srv6 key prefix:
 
```
/vnf-agent/<agent-label>/config/vpp/srv6/v2/
```

### Configuring Local SIDs
The local SID can be configured using this key:
```
/vnf-agent/<agent-label>/config/vpp/srv6/v2/localsid/<SID>
```
where ```<SID>``` (Segment ID) is a unique ID of local sid and it must be an valid IPv6 address. The SID in NB key must
be the same as in the json configuration (value of NB key-value pair). 

### Configuring Policy
The segment routing policy can be configured using this key:
```
/vnf-agent/<agent-label>/config/vpp/srv6/v2/policy/<bsid>
```
where ```<bsid>``` is  unique binding SID of the policy. As any other SRv6 SID it must be an valid IPv6 address. Also 
the SID in NB key must be the same as in the json configuration (value of NB key-value pair).\
The policy can have defined inside value multiple segment lists. The VPP implementation doesn't allow to have policy 
without at least one segment list. Therefore inserting(updating with) policy that has not defined at least one segment 
list will fail (value can be written in ETCD, but its application to VPP will fail as validation error).

### Configuring Steering
The steering (the VPP's policy for steering traffic into SR policy) can be configured using this key:
```
/vnf-agent/<agent-label>/config/vpp/srv6/v2/steering/<name>
```
where ```<name>``` is a unique name of steering.

## Telemetry

The `telemetry` plugin is a core Agent Plugin for exporting telemetry statistics from the VPP to the Prometheus. Statistics are published via registry path `/vpp` on port `9191` and updated every 30 seconds.

**Exported data**

- VPP runtime (`show runtime`)

```bash
                 Name                 State         Calls          Vectors        Suspends         Clocks       Vectors/Call
    acl-plugin-fa-cleaner-process  event wait                0               0               1          4.24e4            0.00
    api-rx-from-ring                any wait                 0               0              18          2.92e5            0.00
    avf-process                    event wait                0               0               1          1.18e4            0.00
    bfd-process                    event wait                0               0               1          1.21e4            0.00
    cdp-process                     any wait                 0               0               1          1.34e5            0.00
    dhcp-client-process             any wait                 0               0               1          4.88e3            0.00
    dns-resolver-process            any wait                 0               0               1          5.88e3            0.00
    fib-walk                        any wait                 0               0               4          1.67e4            0.00
    flow-report-process             any wait                 0               0               1          3.19e3            0.00
    flowprobe-timer-process         any wait                 0               0               1          1.40e4            0.00
    igmp-timer-process             event wait                0               0               1          1.29e4            0.00
    ikev2-manager-process           any wait                 0               0               7          4.58e3            0.00
    ioam-export-process             any wait                 0               0               1          3.49e3            0.00
    ip-route-resolver-process       any wait                 0               0               1          7.07e3            0.00
    ip4-reassembly-expire-walk      any wait                 0               0               1          3.92e3            0.00
    ip6-icmp-neighbor-discovery-ev  any wait                 0               0               7          4.78e3            0.00
    ip6-reassembly-expire-walk      any wait                 0               0               1          5.16e3            0.00
    l2fib-mac-age-scanner-process  event wait                0               0               1          4.57e3            0.00
    lacp-process                   event wait                0               0               1          2.46e5            0.00
    lisp-retry-service              any wait                 0               0               4          1.05e4            0.00
    lldp-process                   event wait                0               0               1          6.79e4            0.00
    memif-process                  event wait                0               0               1          1.94e4            0.00
    nat-det-expire-walk               done                   1               0               0          5.68e3            0.00
    nat64-expire-walk              event wait                0               0               1          5.01e7            0.00
    rd-cp-process                   any wait                 0               0          174857          4.04e2            0.00
    send-rs-process                 any wait                 0               0               1          3.22e3            0.00
    startup-config-process            done                   1               0               1          4.99e3            0.00
    udp-ping-process                any wait                 0               0               1          2.65e4            0.00
    unix-cli-127.0.0.1:38288         active                  0               0               9          4.62e8            0.00
    unix-epoll-input                 polling          12735239               0               0          1.14e3            0.00
    vhost-user-process              any wait                 0               0               1          5.66e3            0.00
    vhost-user-send-interrupt-proc  any wait                 0               0               1          1.95e3            0.00
    vpe-link-state-process         event wait                0               0               1          2.27e3            0.00
    vpe-oam-process                 any wait                 0               0               4          1.11e4            0.00
    vxlan-gpe-ioam-export-process   any wait                 0               0               1          4.04e3            0.00
    wildcard-ip4-arp-publisher-pro event wait                0               0               1          5.49e4            0.00

```

Example:

```bash
# HELP vpp_runtime_calls Number of calls
# TYPE vpp_runtime_calls gauge
...
vpp_runtime_calls{agent="agent1",item="unix-epoll-input",thread="",threadID="0"} 7.65806939e+08
...
# HELP vpp_runtime_clocks Number of clocks
# TYPE vpp_runtime_clocks gauge
...
vpp_runtime_clocks{agent="agent1",item="unix-epoll-input",thread="",threadID="0"} 1150
...
# HELP vpp_runtime_suspends Number of suspends
# TYPE vpp_runtime_suspends gauge
...
vpp_runtime_suspends{agent="agent1",item="unix-epoll-input",thread="",threadID="0"} 0
...
# HELP vpp_runtime_vectors Number of vectors
# TYPE vpp_runtime_vectors gauge
...
vpp_runtime_vectors{agent="agent1",item="unix-epoll-input",thread="",threadID="0"} 0
...
# HELP vpp_runtime_vectors_per_call Number of vectors per call
# TYPE vpp_runtime_vectors_per_call gauge
...
vpp_runtime_vectors_per_call{agent="agent1",item="unix-epoll-input",thread="",threadID="0"} 0
...
```

- VPP buffers (`show buffers`)

```bash
 Thread             Name                 Index       Size        Alloc       Free       #Alloc       #Free
      0                       default           0        2048      0           0           0           0
      0                 lacp-ethernet           1         256      0           0           0           0
      0               marker-ethernet           2         256      0           0           0           0
      0                       ip4 arp           3         256      0           0           0           0
      0        ip6 neighbor discovery           4         256      0           0           0           0
      0                  cdp-ethernet           5         256      0           0           0           0
      0                 lldp-ethernet           6         256      0           0           0           0
      0           replication-recycle           7        1024      0           0           0           0
      0                       default           8        2048      0           0           0           0
```

Example:

```bash
# HELP vpp_buffers_alloc Allocated
# TYPE vpp_buffers_alloc gauge
vpp_buffers_alloc{agent="agent1",index="0",item="default",threadID="0"} 0
vpp_buffers_alloc{agent="agent1",index="1",item="lacp-ethernet",threadID="0"} 0
...
# HELP vpp_buffers_free Free
# TYPE vpp_buffers_free gauge
vpp_buffers_free{agent="agent1",index="0",item="default",threadID="0"} 0
vpp_buffers_free{agent="agent1",index="1",item="lacp-ethernet",threadID="0"} 0
...
# HELP vpp_buffers_num_alloc Number of allocated
# TYPE vpp_buffers_num_alloc gauge
vpp_buffers_num_alloc{agent="agent1",index="0",item="default",threadID="0"} 0
vpp_buffers_num_alloc{agent="agent1",index="1",item="lacp-ethernet",threadID="0"} 0
...
# HELP vpp_buffers_num_free Number of free
# TYPE vpp_buffers_num_free gauge
vpp_buffers_num_free{agent="agent1",index="0",item="default",threadID="0"} 0
vpp_buffers_num_free{agent="agent1",index="1",item="lacp-ethernet",threadID="0"} 0
...
# HELP vpp_buffers_size Size of buffer
# TYPE vpp_buffers_size gauge
vpp_buffers_size{agent="agent1",index="0",item="default",threadID="0"} 2048
vpp_buffers_size{agent="agent1",index="1",item="lacp-ethernet",threadID="0"} 256
...
...
```

- VPP memory (`show memory`)

```bash
Thread 0 vpp_main
20071 objects, 14276k of 14771k used, 21k free, 12k reclaimed, 315k overhead, 1048572k capacity
```

Example:

```bash
# HELP vpp_memory_capacity Capacity
# TYPE vpp_memory_capacity gauge
vpp_memory_capacity{agent="agent1",thread="vpp_main",threadID="0"} 1.048572e+09
# HELP vpp_memory_free Free memory
# TYPE vpp_memory_free gauge
vpp_memory_free{agent="agent1",thread="vpp_main",threadID="0"} 4000
# HELP vpp_memory_objects Number of objects
# TYPE vpp_memory_objects gauge
vpp_memory_objects{agent="agent1",thread="vpp_main",threadID="0"} 20331
# HELP vpp_memory_overhead Overhead
# TYPE vpp_memory_overhead gauge
vpp_memory_overhead{agent="agent1",thread="vpp_main",threadID="0"} 319000
# HELP vpp_memory_reclaimed Reclaimed memory
# TYPE vpp_memory_reclaimed gauge
vpp_memory_reclaimed{agent="agent1",thread="vpp_main",threadID="0"} 0
# HELP vpp_memory_total Total memory
# TYPE vpp_memory_total gauge
vpp_memory_total{agent="agent1",thread="vpp_main",threadID="0"} 1.471e+07
# HELP vpp_memory_used Used memory
# TYPE vpp_memory_used gauge
vpp_memory_used{agent="agent1",thread="vpp_main",threadID="0"} 1.4227e+07
```

- VPP node counters (`show node counters`)

```bash
Count                    Node                  Reason
120406            ipsec-output-ip4            IPSec policy protect
120406               esp-encrypt              ESP pkts received
123692             ipsec-input-ip4            IPSEC pkts received
  3286             ip4-icmp-input             unknown type
120406             ip4-icmp-input             echo replies sent
    14             ethernet-input             l3 mac mismatch
   102                arp-input               ARP replies sent
```

Example:

```bash
# HELP vpp_node_counter_count Count
# TYPE vpp_node_counter_count gauge
vpp_node_counter_count{agent="agent1",item="arp-input",reason="ARP replies sent"} 103
vpp_node_counter_count{agent="agent1",item="esp-encrypt",reason="ESP pkts received"} 124669
vpp_node_counter_count{agent="agent1",item="ethernet-input",reason="l3 mac mismatch"} 14
vpp_node_counter_count{agent="agent1",item="ip4-icmp-input",reason="echo replies sent"} 124669
vpp_node_counter_count{agent="agent1",item="ip4-icmp-input",reason="unknown type"} 3358
vpp_node_counter_count{agent="agent1",item="ipsec-input-ip4",reason="IPSEC pkts received"} 128027
vpp_node_counter_count{agent="agent1",item="ipsec-output-ip4",reason="IPSec policy protect"} 124669
```
    
**Configuration file**

The telemetry plugin configuration file allows to change polling interval, or turn the polling off. The `polling-interval` is time in nanoseconds between reads from the VPP. Parameter `disabled` can be set to `true` in order to disable the telemetry plugin.

[acl-model]: https://github.com/ligato/vpp-agent/blob/master/api/models/vpp/acl/acl.proto
[arp-model]: https://github.com/ligato/vpp-agent/blob/master/api/models/vpp/l3/arp.proto
[bd-model]: https://github.com/ligato/vpp-agent/blob/master/api/models/vpp/l2/bridge-domain.proto
[fib-model]: https://github.com/ligato/vpp-agent/blob/master/api/models/vpp/l2/fib.proto
[grpc-tutorial]: ../tutorials/08_grpc-tutorial.md
[interface-config-file]: https://github.com/ligato/vpp-agent/blob/master/plugins/vpp/ifplugin/vpp-ifplugin.conf
[interface-model]: https://github.com/ligato/vpp-agent/blob/master/api/models/vpp/interfaces/interface.proto
[interface-state-model]: https://github.com/ligato/vpp-agent/blob/master/api/models/vpp/interfaces/state.proto
[ipsec-model]: https://github.com/ligato/vpp-agent/blob/master/api/models/vpp/ipsec/ipsec.proto
[key-reference]: ../user-guide/reference.md
[l3-model]: https://github.com/ligato/vpp-agent/blob/master/api/models/vpp/l3/l3.proto
[linux-interface-plugin-guide]: linux-plugins.md#interface-plugin
[punt-model]: https://github.com/ligato/vpp-agent/blob/master/api/models/vpp/punt/punt.proto
[route-model]: https://github.com/ligato/vpp-agent/blob/master/api/models/vpp/l3/route.proto
[src6-model]: https://github.com/ligato/vpp-agent/blob/master/api/models/vpp/srv6/srv6.proto
[xc-model]: https://github.com/ligato/vpp-agent/blob/master/api/models/vpp/l2/xconnect.proto

*[ACL]: Access Control List
*[ARP]: Address Resolution Protocol
*[DHCP]: Dynamic Host Configuration Protocol
*[DPDK]: Data Plane Development Kit
*[FIB]: Forwarding Information Base
*[NAT]: Network Address Translation
*[REST]: Representational State Transfer
*[SA]: Security Association
*[SPD]: Security Policy Database
*[VPP]: Vector Packet Processing
