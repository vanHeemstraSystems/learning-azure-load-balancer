# 100 - Azure Load Balancer

This is a comprehensive reverse engineering breakdown of Azure Load Balancer following the exact same pattern as the Network Security Group (NSG) breakdown.

## What’s Inside

Following the same 8-layer structure:

**Layer 1: Portal View** - What you see vs what’s really happening  
**Layer 2: Components** - Frontend IPs, backend pools, health probes, rules, NAT  
**Layer 3: How It Works** - 5-tuple hashing, connection tracking, session persistence  
**Layer 4: Technology** - Software Load Balancer (SLB), MUXes, Host Agents, VxLAN  
**Layer 5: TCP/IP Mapping** - Layer 3/4 operations, SNAT/DNAT, port translation  
**Layer 6: Packet Flow** - Complete journey from client to backend and back  
**Layer 7: Physical Implementation** - Datacenter architecture, MUX deployment, network fabric  
**Layer 8: Practical Examples** - Web apps, 3-tier architecture, outbound NAT, HA zones

## Key Topics Covered

- Basic vs Standard SKU comparison
- Internal vs Public load balancers
- Health probe mechanics (TCP, HTTP, HTTPS)
- Distribution algorithms (5-tuple, SourceIP, SourceIPProtocol)
- Floating IP / Direct Server Return (DSR)
- SNAT port allocation and exhaustion
- HA Ports for Network Virtual Appliances
- Zone redundancy and availability
- Complete packet encapsulation with VxLAN
- Four detailed real-world examples

The document maps everything to TCP/IP layers just like the NSG document, showing you exactly where load balancing happens in the network stack and how Azure’s distributed Software Load Balancer works under the hood.

# Reverse Engineering Azure Load Balancer

## From Azure Portal to Network Packets - A Complete Breakdown

A comprehensive deconstruction of how Azure Load Balancer works, from what you click in the portal down to how packets are distributed across backend servers.

-----

## Table of Contents

- [Layer 1: What You See (Azure Portal)](#layer-1-what-you-see-azure-portal)
- [Layer 2: Load Balancer Components & Structure](#layer-2-load-balancer-components--structure)
- [Layer 3: How Load Balancing Works](#layer-3-how-load-balancing-works)
- [Layer 4: Under the Hood - The Technology](#layer-4-under-the-hood---the-technology)
- [Layer 5: Mapping to TCP/IP Model](#layer-5-mapping-to-tcpip-model)
- [Layer 6: Packet Processing Flow](#layer-6-packet-processing-flow)
- [Layer 7: Physical Implementation](#layer-7-physical-implementation)
- [Layer 8: Practical Examples](#layer-8-practical-examples)

-----

## Layer 1: What You See (Azure Portal)

### The User Interface View

When you navigate to a Load Balancer in the Azure Portal, here’s what you see:

```
Azure Portal → Resource Groups → Your Load Balancer
├── Overview
│   ├── Essentials (Name, Resource Group, Location, SKU)
│   ├── Frontend IP configuration
│   ├── Backend pools
│   └── Monitoring metrics
├── Frontend IP configuration
├── Backend pools
├── Health probes
├── Load balancing rules
├── Inbound NAT rules
├── Outbound rules
├── Diagnostics settings
├── Insights (monitoring)
└── Properties
```

### Creating a Load Balancer: The Simple View

**Portal Steps**:

1. Click “Create a resource”
1. Search “Load Balancer”
1. Fill in:
- **Name**: `MyLoadBalancer`
- **Region**: `West Europe`
- **Type**: `Public` or `Internal`
- **SKU**: `Basic` or `Standard`
- **Tier**: `Regional` or `Global`
1. Configure Frontend IP (public IP or internal IP)
1. Create Backend pool
1. Add Health probe
1. Create Load balancing rule
1. Click “Create”

**What Just Happened?**

- Azure deployed a distributed load balancing service
- Created virtual IP address (VIP) for frontend
- Configured backend pool for traffic distribution
- Set up health monitoring for backend servers
- Enabled high availability automatically

### The Abstraction

**What Azure Hides From You**:

```
Simple Portal View:
┌──────────────────────────┐
│  Load Balancer           │
│  - Name: MyLB            │
│  - Type: Public          │
│  - Frontend IP: 20.x.x.x │
│  - Backend: 3 VMs        │
└──────────────────────────┘

What's Actually Running:
┌────────────────────────────────────────────────┐
│ Distributed Software Load Balancer (SLB)       │
│ - Running on Azure multiplexers (MUXes)        │
│ - Replicated across availability zones         │
│ - 40+ million flows/second per MUX             │
│ - Sub-millisecond latency added                │
│ - Automatic failover (no single point failure) │
│ - Integrated with Azure SDN fabric             │
│ - Direct Server Return (DSR) capable           │
│ - Flow-based 5-tuple hash distribution         │
│ - Stateful connection tracking                 │
└────────────────────────────────────────────────┘
```

**Key Concept**: Azure Load Balancer isn’t a VM or appliance you manage—it’s a distributed system running on specialized Azure infrastructure.

-----

## Layer 2: Load Balancer Components & Structure

### Anatomy of an Azure Load Balancer

An Azure Load Balancer consists of several interconnected components:

#### 1. Load Balancer Resource

```json
{
  "name": "MyLoadBalancer",
  "location": "westeurope",
  "type": "Microsoft.Network/loadBalancers",
  "sku": {
    "name": "Standard",
    "tier": "Regional"
  },
  "properties": {
    "frontendIPConfigurations": [],
    "backendAddressPools": [],
    "loadBalancingRules": [],
    "probes": [],
    "inboundNatRules": [],
    "outboundRules": []
  }
}
```

**SKU Comparison**:

|Feature               |Basic              |Standard                     |
|----------------------|-------------------|-----------------------------|
|Backend pool size     |Up to 300 instances|Up to 1000 instances         |
|Health probe protocols|HTTP, TCP          |HTTP, HTTPS, TCP             |
|Availability Zones    |Not supported      |Supported                    |
|SLA                   |None               |99.99%                       |
|Security              |Open by default    |Closed by default (needs NSG)|
|Diagnostics           |Limited metrics    |Azure Monitor integration    |
|Cost                  |Free               |Paid (rules + data processed)|
|HA Ports              |Not supported      |Supported                    |
|Global VNet peering   |Not supported      |Supported                    |

**Tier Types**:

- **Regional**: Load balances within a single region
- **Global**: Load balances across regions (Standard SKU only)

#### 2. Frontend IP Configuration

The **Frontend IP** is what clients connect to—your load balancer’s address.

**Public Load Balancer**:

```json
{
  "name": "PublicFrontend",
  "properties": {
    "privateIPAllocationMethod": null,
    "publicIPAddress": {
      "id": "/subscriptions/.../publicIPAddresses/MyPublicIP"
    },
    "publicIPPrefix": null
  }
}
```

**Internal Load Balancer**:

```json
{
  "name": "InternalFrontend",
  "properties": {
    "privateIPAllocationMethod": "Static",
    "privateIPAddress": "10.0.1.100",
    "subnet": {
      "id": "/subscriptions/.../subnets/AppSubnet"
    }
  }
}
```

**Multiple Frontend IPs**:

```
Single Load Balancer with Multiple Frontends:
Frontend 1: 20.50.60.10 (Public) → Backend Pool 1 (Web servers)
Frontend 2: 20.50.60.11 (Public) → Backend Pool 2 (API servers)
Frontend 3: 10.0.1.100 (Internal) → Backend Pool 3 (Database)

One load balancer, multiple services!
```

**Key Points**:

- Public LB: Uses Public IP addresses
- Internal LB: Uses Private IP from VNet subnet
- Can have multiple frontend IPs per load balancer
- Each frontend can have different rules

#### 3. Backend Pool

The **Backend Pool** contains the resources that receive distributed traffic.

**Structure**:

```json
{
  "name": "WebServersBackendPool",
  "properties": {
    "backendIPConfigurations": [
      {
        "id": "/subscriptions/.../networkInterfaces/VM1-NIC/ipConfigurations/ipconfig1"
      },
      {
        "id": "/subscriptions/.../networkInterfaces/VM2-NIC/ipConfigurations/ipconfig1"
      },
      {
        "id": "/subscriptions/.../networkInterfaces/VM3-NIC/ipConfigurations/ipconfig1"
      }
    ]
  }
}
```

**What Can Be in a Backend Pool?**

```
Supported Backend Types:
├── Virtual Machines (individual VMs)
├── Virtual Machine Scale Sets (VMSS)
├── Availability Sets
├── IP addresses (Standard SKU only)
└── Virtual Machine NICs

Standard SKU supports mixing types in same pool
```

**Backend Pool with IP Addresses** (Standard SKU):

```json
{
  "name": "MixedBackendPool",
  "properties": {
    "loadBalancerBackendAddresses": [
      {
        "name": "VM1",
        "properties": {
          "ipAddress": "10.0.1.4",
          "virtualNetwork": {
            "id": "/subscriptions/.../virtualNetworks/MyVNet"
          }
        }
      },
      {
        "name": "OnPremServer",
        "properties": {
          "ipAddress": "192.168.1.50",
          "virtualNetwork": {
            "id": "/subscriptions/.../virtualNetworks/MyVNet"
          }
        }
      }
    ]
  }
}
```

**Backend Pool Considerations**:

- All backends should be in same VNet (or peered VNets)
- Can span availability zones (Standard SKU)
- Health probe determines which backends receive traffic
- Unhealthy backends automatically removed from rotation

#### 4. Health Probes

**Health Probes** continuously monitor backend health to determine which servers can receive traffic.

**TCP Probe**:

```json
{
  "name": "TCP-Probe-80",
  "properties": {
    "protocol": "Tcp",
    "port": 80,
    "intervalInSeconds": 15,
    "numberOfProbes": 2
  }
}
```

**HTTP Probe**:

```json
{
  "name": "HTTP-Probe",
  "properties": {
    "protocol": "Http",
    "port": 80,
    "requestPath": "/health",
    "intervalInSeconds": 15,
    "numberOfProbes": 2
  }
}
```

**HTTPS Probe** (Standard SKU only):

```json
{
  "name": "HTTPS-Probe",
  "properties": {
    "protocol": "Https",
    "port": 443,
    "requestPath": "/api/health",
    "intervalInSeconds": 5,
    "numberOfProbes": 2
  }
}
```

**Probe Properties Explained**:

|Property         |Purpose                  |Values          |Example|
|-----------------|-------------------------|----------------|-------|
|protocol         |Type of check            |TCP, HTTP, HTTPS|HTTP   |
|port             |Port to probe            |1-65535         |80     |
|requestPath      |HTTP(S) path             |URL path        |/health|
|intervalInSeconds|Probe frequency          |5-300 seconds   |15     |
|numberOfProbes   |Failures before unhealthy|1-100           |2      |

**How Health Probes Work**:

```
Healthy State:
┌──────────────────────────────────┐
│ Load Balancer                     │
│                                   │
│ Every 15 seconds:                 │
│ ├─ Probe → VM1:80/health → 200 ✓ │
│ ├─ Probe → VM2:80/health → 200 ✓ │
│ └─ Probe → VM3:80/health → 200 ✓ │
│                                   │
│ All backends: HEALTHY             │
│ Traffic distributed to all 3      │
└──────────────────────────────────┘

Failure Detected:
┌──────────────────────────────────┐
│ T+0s:  Probe → VM2 → 200 ✓       │
│ T+15s: Probe → VM2 → Timeout ✗   │
│ T+30s: Probe → VM2 → Timeout ✗   │
│                                   │
│ After 2 consecutive failures:     │
│ VM2 marked UNHEALTHY              │
│ Traffic now only to VM1, VM3      │
└──────────────────────────────────┘

Recovery:
┌──────────────────────────────────┐
│ T+45s: Probe → VM2 → 200 ✓       │
│ T+60s: Probe → VM2 → 200 ✓       │
│                                   │
│ After 2 consecutive successes:    │
│ VM2 marked HEALTHY                │
│ Traffic resumes to all 3          │
└──────────────────────────────────┘
```

**Probe Response Requirements**:

```
TCP Probe:
- Success: TCP handshake completes
- Failure: Connection refused or timeout

HTTP/HTTPS Probe:
- Success: HTTP 200 OK within timeout
- Failure: Any other status code (404, 500, etc.) or timeout
- Timeout: Default 30 seconds (not configurable)
```

**Best Practices**:

```
✓ Use HTTP/HTTPS probes for web applications
  (More accurate than TCP)

✓ Create dedicated health endpoint
  Example: /health or /api/health
  
✓ Health endpoint should check:
  - Application can respond
  - Dependencies available (database, etc.)
  - Critical services running

✗ Don't probe root path (/)
  (Might be cached, not accurate)

✗ Don't probe paths requiring authentication
  (Will always fail)

Example Health Endpoint (Node.js):
app.get('/health', async (req, res) => {
  try {
    // Check database connection
    await db.ping();
    
    // Check critical service
    const serviceOk = await checkCriticalService();
    
    if (serviceOk) {
      res.status(200).send('Healthy');
    } else {
      res.status(503).send('Service Unavailable');
    }
  } catch (error) {
    res.status(503).send('Unhealthy');
  }
});
```

#### 5. Load Balancing Rules

**Load Balancing Rules** define how traffic is distributed from frontend to backend.

**Basic Rule Structure**:

```json
{
  "name": "HTTP-Rule",
  "properties": {
    "frontendIPConfiguration": {
      "id": "/subscriptions/.../frontendIPConfigurations/PublicFrontend"
    },
    "backendAddressPool": {
      "id": "/subscriptions/.../backendAddressPools/WebServersPool"
    },
    "probe": {
      "id": "/subscriptions/.../probes/HTTP-Probe"
    },
    "protocol": "Tcp",
    "frontendPort": 80,
    "backendPort": 80,
    "enableFloatingIP": false,
    "idleTimeoutInMinutes": 4,
    "loadDistribution": "Default",
    "enableTcpReset": true,
    "disableOutboundSnat": false
  }
}
```

**Rule Properties Explained**:

**Protocol**:

- `Tcp`: TCP traffic (most common)
- `Udp`: UDP traffic
- `All`: All IP protocols (HA Ports)

**Port Mapping**:

```
frontendPort → backendPort

Example 1: Same Port
Frontend: 80 → Backend: 80
(Client connects to LB:80, forwarded to backend:80)

Example 2: Port Translation
Frontend: 80 → Backend: 8080
(Client connects to LB:80, forwarded to backend:8080)
```

**Load Distribution Algorithms**:

```
1. Default (5-tuple hash):
   Hash(SourceIP, SourcePort, DestIP, DestPort, Protocol)
   
   Properties:
   - Same client might hit different backends
   - Best for stateless applications
   - Maximum distribution
   
   Example:
   Request 1: Client 1.2.3.4:5000 → LB → VM1
   Request 2: Client 1.2.3.4:5001 → LB → VM2
   (Different source port = different backend)

2. SourceIP (2-tuple hash):
   Hash(SourceIP, DestIP)
   
   Properties:
   - Same client always hits same backend
   - Session persistence (sticky sessions)
   - Good for stateful applications
   
   Example:
   Request 1: Client 1.2.3.4:5000 → LB → VM1
   Request 2: Client 1.2.3.4:5001 → LB → VM1
   (Same source IP = same backend)

3. SourceIPProtocol (3-tuple hash):
   Hash(SourceIP, DestIP, Protocol)
   
   Properties:
   - Per-protocol stickiness
   - Same source IP + protocol → same backend
   
   Example:
   TCP Request: Client 1.2.3.4 → LB → VM1
   UDP Request: Client 1.2.3.4 → LB → VM2
   (Different protocol = might be different backend)
```

**Visual Distribution Comparison**:

```
Default (5-tuple):
Client 1.2.3.4
├─ Request 1 (port 50000) → VM1
├─ Request 2 (port 50001) → VM2
├─ Request 3 (port 50002) → VM3
└─ Request 4 (port 50003) → VM1

SourceIP (2-tuple):
Client 1.2.3.4
├─ Request 1 (port 50000) → VM1
├─ Request 2 (port 50001) → VM1 ← Same backend!
├─ Request 3 (port 50002) → VM1 ← Same backend!
└─ Request 4 (port 50003) → VM1 ← Same backend!
```

**Floating IP (Direct Server Return)**:

```
enableFloatingIP: false (Default)
Client → LB (VIP: 20.50.60.10:80) → Backend (DIP: 10.0.1.4:80)

Backend sees:
  Source: 20.50.60.10 (LB's IP after SNAT)
  Destination: 10.0.1.4:80 (Backend's actual IP)

enableFloatingIP: true (DSR mode)
Client → LB (VIP: 20.50.60.10:80) → Backend (10.0.1.4 with VIP 20.50.60.10:80)

Backend sees:
  Source: Client's IP
  Destination: 20.50.60.10:80 (VIP, must be configured on backend)

Use cases for DSR:
- SQL AlwaysOn Availability Groups
- Very high throughput scenarios
- When backend needs to see client IP
```

**TCP Reset**:

```
enableTcpReset: true

When backend becomes unhealthy during active connection:
1. Client has open connection to LB
2. Backend fails health probe
3. LB sends TCP RST to client
4. Client knows to reconnect
5. New connection goes to healthy backend

enableTcpReset: false
- Client connection times out (slower)
- User experience degraded
```

**Idle Timeout**:

```
idleTimeoutInMinutes: 4-30 minutes (default: 4)

Scenario:
T+0min: Client establishes TCP connection
T+1min: Last packet sent
T+5min: No activity for 4 minutes → LB closes connection

Use cases:
- Short timeout (4-10 min): Web applications
- Long timeout (20-30 min): Long-running connections, SFTP, SSH
```

#### 6. Inbound NAT Rules

**Inbound NAT Rules** provide direct access to specific backend instances.

**Use Case**: SSH/RDP to individual VMs behind load balancer

```json
{
  "name": "SSH-to-VM1",
  "properties": {
    "frontendIPConfiguration": {
      "id": "/subscriptions/.../frontendIPConfigurations/PublicFrontend"
    },
    "protocol": "Tcp",
    "frontendPort": 50001,
    "backendPort": 22,
    "enableFloatingIP": false,
    "idleTimeoutInMinutes": 4,
    "enableTcpReset": true,
    "backendIPConfiguration": {
      "id": "/subscriptions/.../networkInterfaces/VM1-NIC/ipConfigurations/ipconfig1"
    }
  }
}
```

**Multiple NAT Rules Example**:

```
Load Balancer Frontend: 20.50.60.10

NAT Rules:
├─ Port 50001 → VM1:22 (SSH to VM1)
├─ Port 50002 → VM2:22 (SSH to VM2)
├─ Port 50003 → VM3:22 (SSH to VM3)
├─ Port 33891 → VM1:3389 (RDP to VM1)
├─ Port 33892 → VM2:3389 (RDP to VM2)
└─ Port 33893 → VM3:3389 (RDP to VM3)

Access:
ssh admin@20.50.60.10 -p 50001  # Connects to VM1
ssh admin@20.50.60.10 -p 50002  # Connects to VM2
ssh admin@20.50.60.10 -p 50003  # Connects to VM3
```

**NAT Rule vs Load Balancing Rule**:

```
Load Balancing Rule:
Frontend:80 → Backend Pool (3 VMs)
Traffic distributed across all healthy backends

NAT Rule:
Frontend:50001 → Specific VM1:22
Traffic always goes to specific backend
```

**Inbound NAT Pools** (for VMSS):

```json
{
  "name": "SSH-Pool",
  "properties": {
    "frontendIPConfiguration": {
      "id": "/subscriptions/.../frontendIPConfigurations/PublicFrontend"
    },
    "protocol": "Tcp",
    "frontendPortRangeStart": 50000,
    "frontendPortRangeEnd": 50099,
    "backendPort": 22
  }
}
```

```
VMSS with 5 instances:
Instance 0: 20.50.60.10:50000 → VM0:22
Instance 1: 20.50.60.10:50001 → VM1:22
Instance 2: 20.50.60.10:50002 → VM2:22
Instance 3: 20.50.60.10:50003 → VM3:22
Instance 4: 20.50.60.10:50004 → VM4:22

VMSS scales to 10 instances:
Automatically assigns ports 50005-50009
```

#### 7. Outbound Rules

**Outbound Rules** control how backend VMs connect to the internet.

**Default Outbound Behavior**:

```
Basic SKU:
- Automatic outbound internet access
- Uses load balancer's public IP for SNAT
- No configuration needed

Standard SKU:
- NO automatic outbound access (secure by default)
- Must explicitly configure:
  1. Outbound rules, OR
  2. NAT Gateway, OR
  3. Public IP on VM, OR
  4. Load balancing rule with outbound enabled
```

**Explicit Outbound Rule**:

```json
{
  "name": "OutboundRule",
  "properties": {
    "frontendIPConfigurations": [
      {
        "id": "/subscriptions/.../frontendIPConfigurations/PublicFrontend"
      }
    ],
    "backendAddressPool": {
      "id": "/subscriptions/.../backendAddressPools/WebServersPool"
    },
    "protocol": "All",
    "enableTcpReset": true,
    "idleTimeoutInMinutes": 4,
    "allocatedOutboundPorts": 1024
  }
}
```

**SNAT Port Allocation**:

```
Problem: Port Exhaustion
- Each outbound connection needs unique (IP, Port) tuple
- 64,000 ports per IP address
- Multiple backend VMs share frontend IP

Example:
Frontend IP: 20.50.60.10
Backend Pool: 10 VMs

Without outbound rule:
Default SNAT ports per VM = 64,000 / 10 = 6,400 ports

With outbound rule (allocatedOutboundPorts: 1024):
Each VM gets exactly 1,024 ports
More predictable, prevents one VM from using all ports
```

**Outbound Rule Best Practices**:

```
✓ Use dedicated frontend IP for outbound
  (Separate from inbound traffic)

✓ Use NAT Gateway instead (better solution)
  - More scalable
  - 64,000 ports per IP
  - Can use multiple IPs
  - Recommended for Standard SKU

✓ Monitor SNAT port usage
  - Watch for port exhaustion
  - Metric: "SNAT Connection Count"

Example Architecture:
┌─────────────────────────────────┐
│ Load Balancer                    │
│ ├─ Frontend 1 (Inbound traffic)  │
│ │  20.50.60.10                   │
│ │  ↓                             │
│ │  Backend Pool → VMs            │
│ │                                │
│ └─ Frontend 2 (Outbound traffic) │
│    20.50.60.11                   │
│    ↑                             │
│    VMs → Internet                │
└─────────────────────────────────┘
```

#### 8. HA Ports (Standard SKU Only)

**HA Ports** allow load balancing ALL ports simultaneously.

```json
{
  "name": "HA-Ports-Rule",
  "properties": {
    "frontendIPConfiguration": {
      "id": "/subscriptions/.../frontendIPConfigurations/InternalFrontend"
    },
    "backendAddressPool": {
      "id": "/subscriptions/.../backendAddressPools/NVAPool"
    },
    "probe": {
      "id": "/subscriptions/.../probes/TCP-Probe"
    },
    "protocol": "All",
    "frontendPort": 0,
    "backendPort": 0
  }
}
```

**HA Ports Use Case - Network Virtual Appliances**:

```
Without HA Ports (Limited):
Rule 1: Port 80 → Backend Pool
Rule 2: Port 443 → Backend Pool
Rule 3: Port 22 → Backend Pool
... (Need rule for every port)

With HA Ports (All traffic):
Rule 1: ALL Ports → Backend Pool
(Single rule handles everything)

Architecture:
VNet Traffic
    ↓
Internal Load Balancer (HA Ports)
    ↓
Backend Pool (Firewall NVAs)
    ├─ NVA1 (Active)
    └─ NVA2 (Standby)
```

**HA Ports Limitations**:

- Internal Load Balancer only (not public)
- Standard SKU only
- Single rule per frontend IP
- Cannot mix with regular port-specific rules on same frontend

-----

## Layer 3: How Load Balancing Works

### Traffic Distribution Algorithm

Azure Load Balancer uses **5-tuple hash distribution** by default:

```
5-Tuple Hash:
1. Source IP address
2. Source Port
3. Destination IP address
4. Destination Port
5. Protocol (TCP/UDP)

Hash Calculation:
hash = hash_function(src_ip, src_port, dst_ip, dst_port, protocol)
backend = backends[hash % num_backends]
```

**Example Distribution**:

```
Backend Pool: [VM1, VM2, VM3]

Request 1:
Hash(1.2.3.4:50000, 20.50.60.10:80, TCP) = 12345
Backend = 12345 % 3 = 0 → VM1

Request 2:
Hash(1.2.3.4:50001, 20.50.60.10:80, TCP) = 67890
Backend = 67890 % 3 = 0 → VM1
(Different source port, but might still hash to same backend)

Request 3:
Hash(5.6.7.8:50000, 20.50.60.10:80, TCP) = 98765
Backend = 98765 % 3 = 1 → VM2

Request 4:
Hash(1.2.3.4:50000, 20.50.60.10:443, TCP) = 45678
Backend = 45678 % 3 = 0 → VM1
(Different destination port = different hash)
```

### Connection Flow States

Azure Load Balancer maintains connection state:

```
Connection Lifecycle:

1. New Connection:
   Client sends SYN → LB
   LB receives, no existing flow
   
2. Backend Selection:
   Calculate 5-tuple hash
   Check backend health
   Select healthy backend
   Create flow entry
   
3. Flow Entry:
   {
     src_ip: 1.2.3.4,
     src_port: 50000,
     dst_ip: 20.50.60.10,
     dst_port: 80,
     protocol: TCP,
     backend: VM1 (10.0.1.4),
     state: ESTABLISHED,
     last_packet: timestamp,
     idle_timeout: 4 minutes
   }
   
4. Subsequent Packets:
   All packets matching 5-tuple → same backend (VM1)
   State table lookup (fast path)
   No rehashing needed
   
5. Connection Termination:
   FIN packets exchanged → flow closed
   OR
   Idle timeout reached → flow expired
   OR
   TCP RST received → flow closed immediately
```

### Symmetric vs Asymmetric Routing

**Symmetric Routing** (Default):

```
Request Path:
Client → LB → Backend VM

Response Path:
Backend VM → LB → Client

LB sees both directions
Maintains flow state accurately
```

**Asymmetric Routing** (DSR/Floating IP):

```
Request Path:
Client → LB → Backend VM

Response Path:
Backend VM → Client (bypasses LB!)

Benefits:
- Reduced LB load
- Lower latency for large responses
- Required for certain applications (SQL AlwaysOn)

Requirements:
- Backend must have VIP configured on loopback
- Backend must handle routing correctly
```

### Health-Based Distribution

**Active Health Monitoring**:

```
Time: T+0s
Backend Status:
├─ VM1: Healthy (receiving traffic)
├─ VM2: Healthy (receiving traffic)
└─ VM3: Healthy (receiving traffic)

Distribution: 33% / 33% / 33%

Time: T+15s (VM2 probe fails)
Backend Status:
├─ VM1: Healthy (receiving traffic)
├─ VM2: Healthy (still receiving - needs 2 failures)
└─ VM3: Healthy (receiving traffic)

Distribution: Still 33% / 33% / 33%

Time: T+30s (VM2 probe fails again)
Backend Status:
├─ VM1: Healthy (receiving traffic)
├─ VM2: Unhealthy (NO traffic) ✗
└─ VM3: Healthy (receiving traffic)

Distribution: 50% / 0% / 50%

Existing connections to VM2:
- Remain active until idle timeout or TCP RST
- New connections: Only to VM1 and VM3

Time: T+45s (VM2 recovers)
Backend Status:
├─ VM1: Healthy (receiving traffic)
├─ VM2: Recovering (still no traffic - needs 2 successes)
└─ VM3: Healthy (receiving traffic)

Distribution: Still 50% / 0% / 50%

Time: T+60s (VM2 probe succeeds again)
Backend Status:
├─ VM1: Healthy (receiving traffic)
├─ VM2: Healthy (now receiving traffic) ✓
└─ VM3: Healthy (receiving traffic)

Distribution: 33% / 33% / 33%
```

### Session Persistence (Sticky Sessions)

When using **SourceIP distribution mode**:

```
Scenario: E-commerce shopping cart

Request 1 (Add item):
Client: 1.2.3.4:50000
Hash(1.2.3.4, 20.50.60.10) = 12345
Backend: VM1
VM1 session: cart = [item1]

Request 2 (Add another item):
Client: 1.2.3.4:50001 (different port!)
Hash(1.2.3.4, 20.50.60.10) = 12345 (same hash!)
Backend: VM1 (same backend!)
VM1 session: cart = [item1, item2]

Request 3 (Checkout):
Client: 1.2.3.4:50002
Hash(1.2.3.4, 20.50.60.10) = 12345
Backend: VM1
VM1 session: cart = [item1, item2] (still there!)

Benefits:
✓ All requests from same client → same backend
✓ Session state maintained
✓ Shopping cart works correctly

Limitations:
✗ If VM1 becomes unhealthy:
  - Session lost
  - Cart data gone (unless using shared session store)
  
✗ If client IP changes (mobile, VPN):
  - Might go to different backend
  - Session lost

Better Solution:
Use shared session store (Redis, Cosmos DB)
- Any backend can access session
- True statelessness
- Load balancer distribution doesn't matter
```

### Zone Redundancy (Standard SKU)

**Zone-Redundant Load Balancer**:

```
Azure Region (e.g., West Europe)
├── Availability Zone 1
│   ├── LB MUX replicas
│   └── Backend VMs
├── Availability Zone 2
│   ├── LB MUX replicas
│   └── Backend VMs
└── Availability Zone 3
    ├── LB MUX replicas
    └── Backend VMs

Frontend IP: Zone-redundant
Backend Pool: VMs across all zones

If Zone 1 fails:
✓ LB remains available (MUXes in Zone 2, 3)
✓ Traffic to VMs in Zone 2, 3
✓ No downtime
✗ Capacity reduced (33% fewer backends)

SLA: 99.99% uptime
```

**Zonal Load Balancer**:

```
Frontend IP: Pinned to Zone 1
Backend Pool: VMs in Zone 1 only

If Zone 1 fails:
✗ LB unavailable
✗ All traffic stopped
✗ Full outage

Use Case: When you need resources in specific zone
SLA: 99.95% uptime
```

-----

## Layer 4: Under the Hood - The Technology

### Azure Software Load Balancer (SLB)

Azure Load Balancer is implemented using **Microsoft’s Software Load Balancer (SLB)** technology.

#### Architecture Overview

```
Azure Network Infrastructure:
┌────────────────────────────────────────────────────┐
│ Physical Network                                    │
│ ├── Top-of-Rack (ToR) Switches                     │
│ ├── Spine Switches                                 │
│ └── Border Routers                                 │
└────────────────────────────────────────────────────┘
                    ↕
┌────────────────────────────────────────────────────┐
│ Software Load Balancer (SLB) Layer                 │
│ ├── Multiplexers (MUXes) - Stateless              │
│ │   - Distributed across datacenter               │
│ │   - Handle packet encapsulation/decapsulation   │
│ │   - ECMP (Equal Cost Multi-Path) routing        │
│ │                                                  │
│ └── Host Agents                                    │
│     - Run on every compute host                   │
│     - Direct Server Return implementation         │
│     - Local flow state management                 │
└────────────────────────────────────────────────────┘
                    ↕
┌────────────────────────────────────────────────────┐
│ Virtual Machines (Your Workloads)                  │
│ ├── VM1 in Backend Pool                           │
│ ├── VM2 in Backend Pool                           │
│ └── VM3 in Backend Pool                           │
└────────────────────────────────────────────────────┘
```

### Multiplexers (MUXes)

**MUX Role**: Distribute incoming traffic to backend VMs

```
MUX Characteristics:
- Stateless (no connection state stored)
- Horizontally scalable
- 40+ million concurrent flows per MUX
- Sub-millisecond latency added
- Multiple MUXes per region (HA)
- ECMP routing between MUXes

MUX Operations:
1. Receive packet on VIP (Virtual IP)
2. Calculate 5-tuple hash
3. Query control plane for backend selection
4. Encapsulate packet (VxLAN/NVGRE)
5. Forward to appropriate compute host
6. Return traffic: Reverse process
```

**MUX Deployment**:

```
Azure Datacenter:
┌──────────────────────────────────────┐
│ Datacenter Building                   │
│ ├── Rack 1                            │
│ │   ├── MUX VM 1                      │
│ │   └── Compute Servers               │
│ ├── Rack 2                            │
│ │   ├── MUX VM 2                      │
│ │   └── Compute Servers               │
│ └── Rack N                            │
│     ├── MUX VM N                      │
│     └── Compute Servers               │
└──────────────────────────────────────┘

ECMP Routing:
Internet → Border Router
         → Routes to ANY MUX (equal cost)
         → All MUXes announce same VIP
         → Network uses ECMP to distribute
```

### Host Agent (SLB Host Agent)

**Host Agent Role**: Implements load balancing on compute hosts

```
Physical Server Architecture:
┌────────────────────────────────────────┐
│ Physical Server                         │
│                                         │
│ ┌────────────────────────────────────┐ │
│ │ Hyper-V Hypervisor                 │ │
│ │                                    │ │
│ │ ┌────────────────────────────────┐ │ │
│ │ │ SLB Host Agent                 │ │ │
│ │ │ - Flow state management        │ │ │
│ │ │ - Backend health tracking      │ │ │
│ │ │ - Packet encap/decap           │ │ │
│ │ │ - Direct Server Return         │ │ │
│ │ └────────────────────────────────┘ │ │
│ │                                    │ │
│ │ ┌────────────────────────────────┐ │ │
│ │ │ Virtual Switch (vSwitch)       │ │ │
│ │ │ - Packet forwarding            │ │ │
│ │ │ - NSG enforcement              │ │ │
│ │ └────────────────────────────────┘ │ │
│ │                                    │ │
│ │ VMs:                               │ │
│ │ ├── VM1 (Backend pool member)     │ │
│ │ ├── VM2 (Backend pool member)     │ │
│ │ └── VM3 (Other customer)          │ │
│ └────────────────────────────────────┘ │
└────────────────────────────────────────┘
```

**Host Agent Operations**:

```
Inbound Packet Processing:
1. MUX forwards encapsulated packet
2. Host Agent receives packet
3. Decapsulates to get original packet
4. Checks local flow state table
5. If new flow: Store flow state
6. Forward to appropriate VM's vNIC
7. VM processes packet

Outbound Packet Processing:
1. VM sends response packet
2. Host Agent intercepts
3. Checks flow state table
4. Applies SNAT (if needed)
5. Encapsulates packet
6. Forwards to MUX
7. MUX sends to client
```

### Encapsulation Technologies

Azure uses **VxLAN** or **NVGRE** for packet encapsulation:

```
Original Packet:
┌─────────────────────────────────┐
│ Ethernet Header                  │
│ - Src MAC: Client                │
│ - Dst MAC: LB                    │
├─────────────────────────────────┤
│ IP Header                        │
│ - Src IP: 1.2.3.4                │
│ - Dst IP: 20.50.60.10 (VIP)      │
├─────────────────────────────────┤
│ TCP Header                       │
│ - Src Port: 50000                │
│ - Dst Port: 80                   │
├─────────────────────────────────┤
│ Payload                          │
└─────────────────────────────────┘

After MUX Encapsulation (VxLAN):
┌─────────────────────────────────┐
│ Outer Ethernet Header            │
│ - Src MAC: MUX                   │
│ - Dst MAC: Compute Host          │
├─────────────────────────────────┤
│ Outer IP Header                  │
│ - Src IP: MUX IP                 │
│ - Dst IP: Compute Host IP        │
├─────────────────────────────────┤
│ UDP Header (VxLAN)               │
│ - Port 4789                      │
├─────────────────────────────────┤
│ VxLAN Header                     │
│ - VNI: Backend Pool ID           │
│ - Flags                          │
├─────────────────────────────────┤
│ Original Packet (above)          │
│ - Inner Ethernet                 │
│ - Inner IP                       │
│ - Inner TCP                      │
│ - Payload                        │
└─────────────────────────────────┘

At Compute Host:
1. Host Agent strips outer headers
2. Extracts original packet
3. Delivers to VM
```

### Control Plane vs Data Plane

**Control Plane**:

```
Responsibilities:
- Load balancer configuration storage
- Backend health monitoring
- Policy distribution to data plane
- API operations (create, update, delete)

Components:
┌────────────────────────────────┐
│ Azure Resource Manager (ARM)   │
│ - REST API endpoint            │
│ - Authentication               │
└────────────────────────────────┘
          ↓
┌────────────────────────────────┐
│ Network Resource Provider      │
│ - Validates LB operations      │
│ - Stores configuration         │
└────────────────────────────────┘
          ↓
┌────────────────────────────────┐
│ SLB Controller                 │
│ - Manages MUXes                │
│ - Distributes policy           │
│ - Health probe orchestration   │
└────────────────────────────────┘
          ↓
┌────────────────────────────────┐
│ MUXes and Host Agents          │
│ - Receive configuration        │
│ - Implement policies           │
└────────────────────────────────┘
```

**Data Plane**:

```
Responsibilities:
- Actual packet processing
- Flow state management
- Traffic distribution
- Minimal latency

Path:
Client Packet
    ↓
Border Router
    ↓
MUX (hardware-accelerated forwarding)
    ↓
Host Agent (vSwitch fast path)
    ↓
VM
```

### Performance Characteristics

**Latency**:

```
Added Latency by Load Balancer:
- MUX processing: < 1 millisecond
- Host Agent processing: < 0.5 milliseconds
- Total added: < 1.5 milliseconds

Comparison:
Direct VM-to-VM: 0.5 ms
Through Load Balancer: 2 ms (1.5 ms added)

For typical web apps:
Total request latency: 100-500 ms
LB contribution: < 1%
```

**Throughput**:

```
Per-Flow Limits:
- TCP connection: Limited by TCP window size
- Not limited by load balancer

Aggregate Throughput:
- Standard SKU: No published limit
- In practice: Hundreds of Gbps
- Scales horizontally with MUXes

Per-VM Limits:
- Depends on VM size
- Network bandwidth limits apply
- Not load balancer bottleneck
```

**Connection Limits**:

```
Standard SKU:
- Concurrent flows: Millions per load balancer
- New flows/second: 500,000+
- Backend pool: Up to 1,000 instances

Basic SKU:
- Concurrent flows: 10,000 per flow
- New flows/second: ~10,000
- Backend pool: Up to 300 instances
```

### High Availability Implementation

**MUX Redundancy**:

```
Multiple MUXes per VIP:
VIP: 20.50.60.10

Border routers announce VIP via multiple paths:
Path 1: Border Router → MUX1
Path 2: Border Router → MUX2  
Path 3: Border Router → MUX3

ECMP distributes incoming traffic:
- Router hashes packet 5-tuple
- Selects path to MUX
- If MUX fails, routes removed
- Traffic redistributed to healthy MUXes

Failure Scenario:
T+0s: MUX1 fails
T+1s: BGP detects failure
T+2s: Routes via MUX1 withdrawn
T+3s: All traffic via MUX2, MUX3
T+5s: New MUX instance starting
T+60s: New MUX online, routes added

Downtime: 0 seconds (other MUXes handle traffic)
```

**Backend Health Monitoring**:

```
Health Probe Architecture:
┌────────────────────────────────┐
│ SLB Controller                 │
│ - Orchestrates probes          │
│ - Collects health status       │
└────────────────────────────────┘
          ↓
┌────────────────────────────────┐
│ Distributed Probe Agents       │
│ - Multiple agents per backend  │
│ - Send probes every interval   │
│ - Report results to controller │
└────────────────────────────────┘
          ↓
┌────────────────────────────────┐
│ Backend VMs                    │
│ - Respond to probes            │
└────────────────────────────────┘

Failure Detection:
1. Probe fails at T+0s
2. Agent reports to controller
3. Controller waits for numberOfProbes failures
4. After threshold: Backend marked unhealthy
5. Controller updates MUXes and Host Agents
6. New flows don't go to unhealthy backend
7. Existing flows remain until idle timeout (optional TCP RST)

Recovery:
1. Probe succeeds
2. Controller waits for 2 consecutive successes
3. Backend marked healthy
4. Configuration pushed to data plane
5. Backend starts receiving traffic

Typical timeline:
Detection: 30 seconds (2 failures at 15s interval)
Recovery: 30 seconds (2 successes at 15s interval)
```

-----

## Layer 5: Mapping to TCP/IP Model

Let’s map Azure Load Balancer operations to the TCP/IP model:

### TCP/IP Layer Overview

```
┌─────────────────────────────────────────────┐
│ Layer 4: Application Layer                  │
│ - HTTP, FTP, SMTP, DNS                      │
│ LB: ✗ Does NOT inspect application data    │
│     (Azure Application Gateway does this)   │
└─────────────────────────────────────────────┘
                    ↕
┌─────────────────────────────────────────────┐
│ Layer 3: Transport Layer                    │
│ - TCP, UDP                                  │
│ LB: ✓ PRIMARY OPERATION LAYER              │
│     ✓ Port mapping (frontend → backend)    │
│     ✓ Connection state tracking            │
│     ✓ Protocol filtering (TCP/UDP)         │
│     ✓ Port translation                     │
└─────────────────────────────────────────────┘
                    ↕
┌─────────────────────────────────────────────┐
│ Layer 2: Internet Layer                     │
│ - IP (IPv4/IPv6), ICMP                      │
│ LB: ✓ CRITICAL OPERATION LAYER             │
│     ✓ IP address translation (SNAT/DNAT)   │
│     ✓ VIP to DIP mapping                   │
│     ✓ Health probes (ICMP not supported)   │
│     ✓ Routing decisions                    │
└─────────────────────────────────────────────┘
                    ↕
┌─────────────────────────────────────────────┐
│ Layer 1: Network Access Layer               │
│ - Ethernet, MAC addresses, VxLAN            │
│ LB: ✓ Encapsulation layer                  │
│     ✓ VxLAN/NVGRE encapsulation            │
│     ✗ Does NOT filter by MAC address       │
└─────────────────────────────────────────────┘
```

### Layer 1: Network Access Layer

**What Happens Here**:

```
Physical/Data Link Layer:
- Ethernet frames
- MAC addressing
- VxLAN encapsulation
- Physical transmission

LB Role: Encapsulation/Decapsulation
- Packets encapsulated in VxLAN by MUX
- Decapsulated by Host Agent
- MAC addresses rewritten
```

**Frame Structure** (with VxLAN):

```
Outer Frame (MUX → Compute Host):
┌─────────────────────────────────┐
│ Outer Ethernet Header            │
│ - Src MAC: MUX MAC               │
│ - Dst MAC: Compute Host MAC      │
│ - EtherType: 0x0800 (IPv4)       │
└─────────────────────────────────┘

Inner Frame (Original):
┌─────────────────────────────────┐
│ Inner Ethernet Header            │
│ - Src MAC: Client MAC            │
│ - Dst MAC: VM MAC                │
│ - EtherType: 0x0800 (IPv4)       │
└─────────────────────────────────┘

LB operates above this layer for routing decisions
```

### Layer 2: Internet Layer (IP)

**What Happens Here**:

```
IP Layer:
- IP addressing
- Routing
- Fragmentation
- Address translation

LB Role: PRIMARY ADDRESS TRANSLATION
- VIP (Virtual IP) → DIP (Direct IP)
- SNAT (Source NAT) for outbound
- DNAT (Destination NAT) for inbound
```

**IP Packet Transformation**:

```
Inbound Request (Client → Backend):

Original Packet at MUX:
┌─────────────────────────────────┐
│ IP Header                        │
│ - Src IP: 1.2.3.4 (Client)       │
│ - Dst IP: 20.50.60.10 (VIP)      │ ← Frontend IP
│ - Protocol: TCP                  │
│ - TTL: 50                        │
└─────────────────────────────────┘

After LB Processing (at VM):
┌─────────────────────────────────┐
│ IP Header                        │
│ - Src IP: 1.2.3.4 (Client)       │ ← Preserved (or SNAT'd)
│ - Dst IP: 10.0.1.4 (DIP)         │ ← Changed to backend IP
│ - Protocol: TCP                  │
│ - TTL: 49                        │
└─────────────────────────────────┘

Operations:
1. DNAT: VIP → Backend DIP
2. Routing: Determine which backend
3. Optional SNAT: Change source IP
```

**SNAT (Source Network Address Translation)**:

```
Without SNAT (DSR/Floating IP = true):
Client → LB → Backend
Source IP stays: 1.2.3.4 (client)
Backend sees original client IP

Backend Response:
Backend → Client (bypasses LB)
Source IP: VIP (backend must be configured)
Destination IP: 1.2.3.4

With SNAT (Floating IP = false):
Client → LB → Backend
Source IP changed: 20.50.60.10 (LB's IP)
Backend sees LB as client

Backend Response:
Backend → LB → Client
Source IP: 10.0.1.4 (backend)
LB changes to: 20.50.60.10 (VIP)
Then to client: 1.2.3.4
```

**Health Probe Packets**:

```
TCP Health Probe:
┌─────────────────────────────────┐
│ IP Header                        │
│ - Src IP: 168.63.129.16          │ ← Azure LB probe IP
│ - Dst IP: 10.0.1.4 (Backend)     │
│ - Protocol: TCP                  │
└─────────────────────────────────┘

HTTP Health Probe:
┌─────────────────────────────────┐
│ IP Header                        │
│ - Src IP: 168.63.129.16          │ ← Always this IP
│ - Dst IP: 10.0.1.4               │
│ - Protocol: TCP                  │
├─────────────────────────────────┤
│ TCP Header                       │
│ - Src Port: Random               │
│ - Dst Port: 80                   │
├─────────────────────────────────┤
│ HTTP Request                     │
│ GET /health HTTP/1.1             │
│ Host: ...                        │
└─────────────────────────────────┘

Note: NSG must allow 168.63.129.16 or probes fail!
```

### Layer 3: Transport Layer (TCP/UDP)

**What Happens Here**:

```
Transport Layer:
- TCP or UDP
- Port numbers
- Connection management
- Flow control

LB Role: PRIMARY PORT MAPPING
- Frontend port → Backend port
- Connection state tracking (TCP)
- Port translation
```

**TCP Connection through Load Balancer**:

```
Three-Way Handshake:

Step 1: Client → LB (SYN)
┌─────────────────────────────────┐
│ IP: 1.2.3.4 → 20.50.60.10        │
│ TCP: Port 50000 → Port 80        │
│ Flags: SYN                       │
│ Seq: 1000                        │
└─────────────────────────────────┘

Step 2: LB → Backend (SYN, modified)
┌─────────────────────────────────┐
│ IP: 1.2.3.4 → 10.0.1.4           │ ← DNAT applied
│ TCP: Port 50000 → Port 80        │
│ Flags: SYN                       │
│ Seq: 1000                        │
└─────────────────────────────────┘

LB creates flow entry:
{
  client: 1.2.3.4:50000,
  vip: 20.50.60.10:80,
  backend: 10.0.1.4:80,
  state: SYN_SENT,
  timestamp: T0
}

Step 3: Backend → LB (SYN-ACK)
┌─────────────────────────────────┐
│ IP: 10.0.1.4 → 1.2.3.4           │
│ TCP: Port 80 → Port 50000        │
│ Flags: SYN, ACK                  │
│ Seq: 5000, Ack: 1001             │
└─────────────────────────────────┘

Step 4: LB → Client (SYN-ACK, modified)
┌─────────────────────────────────┐
│ IP: 20.50.60.10 → 1.2.3.4        │ ← Source changed to VIP
│ TCP: Port 80 → Port 50000        │
│ Flags: SYN, ACK                  │
│ Seq: 5000, Ack: 1001             │
└─────────────────────────────────┘

LB updates flow:
{
  state: ESTABLISHED,
  last_packet: T1
}

Step 5: Client → LB (ACK)
[Forwarded to backend via flow state lookup]

Connection Established!
All subsequent packets use flow state (fast path)
```

**UDP Load Balancing**:

```
UDP is connectionless, but LB still tracks "flows":

First Packet:
Client: 1.2.3.4:50000 → VIP: 20.50.60.10:53 (DNS)

LB creates flow entry:
{
  5-tuple hash calculated,
  backend selected: 10.0.1.4,
  timeout: idle_timeout (default 4 min)
}

Subsequent packets from same 5-tuple:
→ Same backend (flow affinity)

After idle timeout:
→ Flow entry deleted
→ Next packet might go to different backend

UDP "Connection" Lifecycle:
1. First packet: Create flow, select backend
2. Subsequent packets: Use existing flow
3. Idle timeout: Delete flow
4. Next packet: New flow (might be different backend)
```

**Port Translation**:

```
Frontend Port ≠ Backend Port:

Load Balancing Rule:
frontendPort: 80
backendPort: 8080

Client Request:
┌─────────────────────────────────┐
│ TCP: 1.2.3.4:50000               │
│   → 20.50.60.10:80               │ ← Client connects to port 80
└─────────────────────────────────┘

After LB:
┌─────────────────────────────────┐
│ TCP: 1.2.3.4:50000               │
│   → 10.0.1.4:8080                │ ← Backend receives on port 8080
└─────────────────────────────────┘

Backend Response:
┌─────────────────────────────────┐
│ TCP: 10.0.1.4:8080               │
│   → 1.2.3.4:50000                │
└─────────────────────────────────┘

After LB (to client):
┌─────────────────────────────────┐
│ TCP: 20.50.60.10:80              │ ← Source port changed back to 80
│   → 1.2.3.4:50000                │
└─────────────────────────────────┘

Client sees:
- Connected to port 80 (doesn't know about 8080)
```

**Connection Idle Timeout**:

```
TCP Connection State Tracking:

T+0min: Client sends data
└─ Packet forwarded to backend
└─ Flow last_packet updated

T+2min: Client sends more data
└─ Flow last_packet updated

T+6min: No activity for 4 minutes
└─ Idle timeout reached
└─ Flow entry deleted

T+6min+1s: Client sends data
└─ No flow entry exists!
└─ If enableTcpReset=true: LB sends RST to client
└─ If enableTcpReset=false: Packet dropped

Client behavior:
- With TCP RST: Immediately knows connection dead, can reconnect
- Without RST: Waits for TCP timeout (~several minutes), bad UX

Best Practice:
✓ Enable TCP Reset for better client experience
✓ Use appropriate idle timeout for your application
  - Short (4-10 min): Web apps
  - Long (20-30 min): SSH, long-running connections
```

### Layer 4: Application Layer

**What Happens Here**:

```
Application Layer:
- HTTP, FTP, SMTP, DNS, etc.
- Application protocols
- Payload content

LB Role: TRANSPARENT (doesn't inspect)
- Forwards application data unchanged
- No HTTP header inspection
- No URL routing
- No SSL termination
```

**HTTP Request** (LB doesn’t see inside):

```
TCP Payload:
┌─────────────────────────────────┐
│ GET /api/users HTTP/1.1          │
│ Host: www.example.com            │
│ User-Agent: Mozilla/5.0          │
│ Cookie: session=abc123           │
│ ...                              │
└─────────────────────────────────┘

Load Balancer sees:
✓ IP addresses (Layer 2)
✓ Ports (Layer 3)
✓ TCP protocol (Layer 3)

Load Balancer does NOT see:
✗ HTTP method (GET)
✗ URL path (/api/users)
✗ Host header
✗ Cookies
✗ Any application data

This is Layer 4 load balancing!

For Layer 7 (application-aware) load balancing:
→ Use Azure Application Gateway
   - HTTP header inspection
   - URL-based routing
   - Cookie-based session affinity
   - SSL termination
   - WAF (Web Application Firewall)
```

**Comparison: Layer 4 vs Layer 7**:

```
Layer 4 Load Balancer (Azure LB):
┌─────────────────────────────────┐
│ Client → LB                      │
│   TCP connection established     │
│   LB forwards TCP packets        │
│   Backend sees TCP connection    │
│                                  │
│ Advantages:                      │
│ ✓ Very fast (no app inspection) │
│ ✓ Protocol agnostic              │
│ ✓ Low latency                    │
│ ✓ Works with any TCP/UDP         │
│                                  │
│ Limitations:                     │
│ ✗ No URL routing                 │
│ ✗ No cookie-based persistence    │
│ ✗ No SSL termination             │
│ ✗ No HTTP header manipulation    │
└─────────────────────────────────┘

Layer 7 Load Balancer (App Gateway):
┌─────────────────────────────────┐
│ Client → LB                      │
│   LB terminates HTTP connection  │
│   LB inspects HTTP request       │
│   LB opens new connection to     │
│   backend based on content       │
│                                  │
│ Advantages:                      │
│ ✓ URL-based routing              │
│ ✓ Cookie-based persistence       │
│ ✓ SSL termination                │
│ ✓ HTTP header manipulation       │
│ ✓ WAF integration                │
│                                  │
│ Limitations:                     │
│ ✗ HTTP/HTTPS only                │
│ ✗ Higher latency                 │
│ ✗ More expensive                 │
└─────────────────────────────────┘
```

-----

## Layer 6: Packet Processing Flow

### Complete Packet Journey

Let’s trace a complete HTTP request through Azure Load Balancer:

#### Scenario Setup

```
Client (Internet): 1.2.3.4
     ↓
Azure Public IP: 20.50.60.10 (VIP)
     ↓
Load Balancer: Standard SKU
  ├─ Frontend: 20.50.60.10
  ├─ Backend Pool: 3 VMs
  │  ├─ VM1: 10.0.1.4 (Healthy)
  │  ├─ VM2: 10.0.1.5 (Healthy)
  │  └─ VM3: 10.0.1.6 (Unhealthy)
  ├─ Health Probe: HTTP /health port 80
  └─ Rule: Port 80 → Port 80 (5-tuple distribution)
```

### Inbound Request Flow (Client → Backend)

**Step 1: Client Initiates Connection**

```
Application: Browser
Action: User types http://20.50.60.10/

DNS (if domain name):
1. Browser queries DNS
2. DNS returns: 20.50.60.10
3. Browser creates HTTP request

TCP Connection Initiation:
1. Browser creates TCP SYN packet
2. Chooses ephemeral port: 50000

Packet Created:
┌─────────────────────────────────┐
│ Ethernet Header                  │
│ - Src MAC: Client's router MAC   │
│ - Dst MAC: ISP MAC               │
├─────────────────────────────────┤
│ IP Header                        │
│ - Src IP: 1.2.3.4                │
│ - Dst IP: 20.50.60.10            │
│ - Protocol: TCP (6)              │
│ - TTL: 64                        │
├─────────────────────────────────┤
│ TCP Header                       │
│ - Src Port: 50000                │
│ - Dst Port: 80                   │
│ - Flags: SYN                     │
│ - Seq: 1000 (random)             │
│ - Window: 65535                  │
└─────────────────────────────────┘
```

**Step 2: Packet Traverses Internet**

```
Path:
Client → ISP → Internet Backbone → Microsoft Network Edge → Azure

At Microsoft Network Edge:
1. Packet arrives at border router
2. Router checks destination: 20.50.60.10
3. Routes via BGP to Azure datacenter
4. Enters Azure fabric network
```

**Step 3: Border Router → MUX (via ECMP)**

```
Border Router Decision:
- Destination: 20.50.60.10 (VIP)
- Multiple MUXes announce this VIP
- ECMP routes available:
  ├─ Path 1: → MUX1 (cost: 10)
  ├─ Path 2: → MUX2 (cost: 10)
  └─ Path 3: → MUX3 (cost: 10)

ECMP Hash:
hash = hash(src_ip, src_port, dst_ip, dst_port, proto)
hash(1.2.3.4, 50000, 20.50.60.10, 80, TCP) = 12345
path = 12345 % 3 = 0
Selected: Path 1 → MUX1

Packet routed to MUX1
```

**Step 4: MUX Processing**

```
MUX1 Receives Packet:

1. Extract packet fields:
   - Src: 1.2.3.4:50000
   - Dst: 20.50.60.10:80
   - Protocol: TCP
   - Flags: SYN (new connection)

2. Lookup VIP configuration:
   VIP: 20.50.60.10
   Backend Pool: [10.0.1.4, 10.0.1.5, 10.0.1.6]
   Health Status: [Healthy, Healthy, Unhealthy]
   Distribution: 5-tuple hash

3. Calculate backend selection:
   Available backends: [10.0.1.4, 10.0.1.5]  (VM3 excluded)
   hash = hash(1.2.3.4, 50000, 20.50.60.10, 80, TCP)
   hash = 67890
   backend_index = 67890 % 2 = 0
   Selected backend: 10.0.1.4 (VM1)

4. Lookup backend location:
   Backend IP: 10.0.1.4
   Physical host: ComputeHost-7
   Host IP: 172.16.10.20 (internal datacenter IP)

5. Encapsulate packet (VxLAN):
   
   Original Packet:
   [IP: 1.2.3.4 → 20.50.60.10 | TCP: 50000 → 80 | SYN]
   
   Encapsulated Packet:
   ┌─────────────────────────────────┐
   │ Outer Ethernet                   │
   │ - Src: MUX1 MAC                  │
   │ - Dst: ComputeHost-7 MAC         │
   ├─────────────────────────────────┤
   │ Outer IP                         │
   │ - Src: MUX1 IP (172.16.5.10)     │
   │ - Dst: Host IP (172.16.10.20)    │
   ├─────────────────────────────────┤
   │ UDP Header                       │
   │ - Port: 4789 (VxLAN)             │
   ├─────────────────────────────────┤
   │ VxLAN Header                     │
   │ - VNI: BackendPool-ID            │
   │ - Metadata: Backend=10.0.1.4     │
   ├─────────────────────────────────┤
   │ Inner Ethernet (original)        │
   │ Inner IP (original)              │
   │ - Src: 1.2.3.4                   │
   │ - Dst: 20.50.60.10               │
   │ Inner TCP (original)             │
   │ - Src Port: 50000                │
   │ - Dst Port: 80                   │
   │ - Flags: SYN                     │
   └─────────────────────────────────┘

6. Forward to compute host
   Packet sent on Azure datacenter fabric network
```

**Step 5: Compute Host Processing**

```
ComputeHost-7 Receives Packet:

1. Physical NIC receives packet
   
2. vSwitch inspects outer headers:
   - Dst IP: 172.16.10.20 (this host)
   - UDP Port: 4789 (VxLAN)
   
3. SLB Host Agent triggered:
   - Decapsulates VxLAN
   - Extracts original packet
   - Reads metadata: Backend=10.0.1.4
   
4. Extract original packet:
   [IP: 1.2.3.4 → 20.50.60.10 | TCP: 50000 → 80 | SYN]

5. Apply DNAT (Destination NAT):
   Before: Dst IP = 20.50.60.10 (VIP)
   After:  Dst IP = 10.0.1.4 (Backend DIP)
   
   Transformed Packet:
   ┌─────────────────────────────────┐
   │ IP Header                        │
   │ - Src: 1.2.3.4                   │ ← Preserved
   │ - Dst: 10.0.1.4                  │ ← Changed to DIP
   │ - Protocol: TCP                  │
   ├─────────────────────────────────┤
   │ TCP Header                       │
   │ - Src Port: 50000                │
   │ - Dst Port: 80                   │
   │ - Flags: SYN                     │
   └─────────────────────────────────┘

6. Create flow state entry:
   Flow Table:
   {
     5-tuple: (1.2.3.4:50000, 20.50.60.10:80, TCP),
     backend: 10.0.1.4,
     state: SYN_SENT,
     timestamp: T0,
     idle_timeout: 240 seconds
   }

7. NSG Evaluation (if attached):
   - Check NSG rules
   - Allow inbound port 80 from Internet
   - Packet passes

8. Forward to VM1's vNIC:
   - vSwitch internal forwarding
   - Packet delivered to VM1
```

**Step 6: VM1 (Backend) Processing**

```
VM1 Operating System:

1. Network driver receives packet
   
2. IP stack processing:
   - Destination IP: 10.0.1.4 ✓ (my IP)
   - Protocol: TCP ✓
   
3. TCP stack processing:
   - Destination Port: 80 ✓ (nginx listening)
   - Flags: SYN (new connection)
   - Create TCP control block (TCB)
   
4. TCP Three-Way Handshake (Server Side):
   Received: SYN
   Send: SYN-ACK
   
   Create Response Packet:
   ┌─────────────────────────────────┐
   │ IP Header                        │
   │ - Src: 10.0.1.4                  │
   │ - Dst: 1.2.3.4                   │
   │ - Protocol: TCP                  │
   ├─────────────────────────────────┤
   │ TCP Header                       │
   │ - Src Port: 80                   │
   │ - Dst Port: 50000                │
   │ - Flags: SYN, ACK                │
   │ - Seq: 5000 (random)             │
   │ - Ack: 1001 (received seq + 1)   │
   └─────────────────────────────────┘

5. Packet enters VM's network stack for transmission
```

### Outbound Response Flow (Backend → Client)

**Step 7: VM1 Sends Response**

```
VM1 Transmits SYN-ACK:

1. Packet leaves VM's network interface
   
2. vSwitch receives packet:
   - Source: 10.0.1.4:80
   - Dest: 1.2.3.4:50000
   
3. SLB Host Agent intercepts:
   
   a) Lookup flow state:
      Search flow table for matching 5-tuple
      Found: Flow from Step 6
      {
        5-tuple: (1.2.3.4:50000, 20.50.60.10:80, TCP),
        backend: 10.0.1.4,
        state: SYN_SENT → ESTABLISHED
      }
   
   b) Apply Reverse NAT (SNAT):
      Before: Src IP = 10.0.1.4 (Backend DIP)
      After:  Src IP = 20.50.60.10 (VIP)
      
      Transformed Packet:
      ┌─────────────────────────────────┐
      │ IP Header                        │
      │ - Src: 20.50.60.10               │ ← Changed to VIP
      │ - Dst: 1.2.3.4                   │
      │ - Protocol: TCP                  │
      ├─────────────────────────────────┤
      │ TCP Header                       │
      │ - Src Port: 80                   │
      │ - Dst Port: 50000                │
      │ - Flags: SYN, ACK                │
      │ - Seq: 5000                      │
      │ - Ack: 1001                      │
      └─────────────────────────────────┘
   
   c) Routing decision:
      Destination: 1.2.3.4 (external)
      Route via: MUX (for consistency)
      
      Note: Could use Direct Server Return (DSR) if configured,
            but default is return via MUX
   
   d) Encapsulate (VxLAN):
      ┌─────────────────────────────────┐
      │ Outer Ethernet                   │
      │ - Src: Host MAC                  │
      │ - Dst: MUX1 MAC                  │
      ├─────────────────────────────────┤
      │ Outer IP                         │
      │ - Src: Host IP (172.16.10.20)    │
      │ - Dst: MUX1 IP (172.16.5.10)     │
      ├─────────────────────────────────┤
      │ UDP (VxLAN)                      │
      ├─────────────────────────────────┤
      │ VxLAN Header                     │
      │ - Flow metadata                  │
      ├─────────────────────────────────┤
      │ Inner Packet (transformed above) │
      │ [IP: 20.50.60.10 → 1.2.3.4]      │
      │ [TCP: 80 → 50000, SYN-ACK]       │
      └─────────────────────────────────┘
   
   e) Send to MUX1
```

**Step 8: MUX1 Returns Response**

```
MUX1 Receives Return Packet:

1. Decapsulate VxLAN:
   - Extract inner packet
   - [IP: 20.50.60.10 → 1.2.3.4 | TCP: SYN-ACK]

2. Routing decision:
   - Destination: 1.2.3.4 (Internet)
   - Route via border router

3. Send to border router:
   - Packet forwarded on Azure fabric
   - No further encapsulation needed
```

**Step 9: Border Router → Internet**

```
Border Router:

1. Receives packet from MUX
2. Destination: 1.2.3.4 (external)
3. BGP routing to internet
4. NAT applied (Azure public IP handling)
5. Packet exits Azure network

Packet on Internet:
┌─────────────────────────────────┐
│ IP Header                        │
│ - Src: 20.50.60.10               │
│ - Dst: 1.2.3.4                   │
├─────────────────────────────────┤
│ TCP: SYN-ACK                     │
└─────────────────────────────────┘
```

**Step 10: Client Receives Response**

```
Client System:

1. Packet arrives from internet
2. TCP stack processes SYN-ACK
3. Connection state: SYN-SENT → ESTABLISHED
4. Sends final ACK:

   ┌─────────────────────────────────┐
   │ IP: 1.2.3.4 → 20.50.60.10        │
   │ TCP: 50000 → 80, ACK             │
   │ Seq: 1001, Ack: 5001             │
   └─────────────────────────────────┘

5. Three-way handshake complete!
6. Connection established
```

### Subsequent HTTP Request (Fast Path)

```
Client Sends HTTP GET:

Packet:
┌─────────────────────────────────┐
│ IP: 1.2.3.4 → 20.50.60.10        │
│ TCP: 50000 → 80, PSH-ACK         │
│ Payload:                         │
│   GET / HTTP/1.1                 │
│   Host: 20.50.60.10              │
│   ...                            │
└─────────────────────────────────┘

At MUX:
1. 5-tuple: (1.2.3.4:50000, 20.50.60.10:80, TCP)
2. Calculate hash: 67890
3. Backend: 10.0.1.4 (same as before) ✓
4. Encapsulate and forward

At Host Agent:
1. Lookup flow state: FOUND ✓
2. Fast path: Direct forwarding
3. No full rule evaluation needed
4. Microsecond latency

At VM1:
1. TCP stack delivers to nginx
2. nginx processes HTTP request
3. Generates HTTP response
4. Sends back through established connection

Response takes reverse path (via flow state)
```

### Health Probe Flow

Parallel to client traffic, health probes continuously run:

```
Every 15 seconds:

SLB Controller → Probe Agent:
"Check health of 10.0.1.4:80"

Probe Packet:
┌─────────────────────────────────┐
│ IP Header                        │
│ - Src: 168.63.129.16             │ ← Azure health probe IP
│ - Dst: 10.0.1.4                  │
├─────────────────────────────────┤
│ TCP: SYN to port 80              │
│ (for TCP probe)                  │
│                                  │
│ OR                               │
│                                  │
│ TCP connection + HTTP GET        │
│ GET /health HTTP/1.1             │
│ (for HTTP probe)                 │
└─────────────────────────────────┘

VM1 Response:
- TCP probe: SYN-ACK (connection succeeds)
- HTTP probe: HTTP 200 OK

Probe Agent → SLB Controller:
"10.0.1.4 is healthy"

If 2 consecutive probes fail:
1. Controller marks backend unhealthy
2. Updates MUX and Host Agent configs
3. New flows don't go to failed backend
4. Existing flows continue until idle timeout
```

-----

## Layer 7: Physical Implementation

### Datacenter Architecture

**Physical Layout**:

```
Azure Region (e.g., West Europe):
┌────────────────────────────────────────┐
│ Multiple Datacenter Buildings           │
│                                         │
│ Building 1 (Availability Zone 1):      │
│ ┌────────────────────────────────────┐ │
│ │ Row 1                              │ │
│ │ ├─ Rack 1                          │ │
│ │ │  ├─ ToR Switch                   │ │
│ │ │  ├─ Server 1                     │ │
│ │ │  │  ├─ MUX VM (virtual)          │ │
│ │ │  │  └─ Customer VMs              │ │
│ │ │  ├─ Server 2-40                  │ │
│ │ │  │  └─ Customer VMs              │ │
│ │ │  └─ Power Distribution Unit      │ │
│ │ ├─ Rack 2-50                       │ │
│ │ └─ Spine Switches                  │ │
│ └────────────────────────────────────┘ │
│                                         │
│ Building 2 (Availability Zone 2)        │
│ Building 3 (Availability Zone 3)        │
│                                         │
│ Edge Infrastructure:                    │
│ ├─ Border Routers                       │
│ ├─ Edge Switches                        │
│ └─ Internet Connectivity                │
└────────────────────────────────────────┘
```

### MUX Physical Deployment

```
MUX Implementation:
- NOT dedicated hardware appliances
- VMs running on standard Azure compute
- Specialized networking VMs
- Distributed across datacenter

Typical Deployment:
┌────────────────────────────────────────┐
│ Datacenter has ~10-100 MUX VMs          │
│                                         │
│ MUX Distribution:                       │
│ ├─ Zone 1: 5-10 MUX VMs                 │
│ ├─ Zone 2: 5-10 MUX VMs                 │
│ └─ Zone 3: 5-10 MUX VMs                 │
│                                         │
│ Each MUX:                               │
│ ├─ Multiple CPU cores (10-20)          │
│ ├─ Memory: 32-64 GB                     │
│ ├─ Network: 25-100 Gbps NICs            │
│ ├─ SR-IOV for performance               │
│ └─ DPDK (Data Plane Development Kit)    │
└────────────────────────────────────────┘

MUX Software Stack:
┌────────────────────────────────────────┐
│ Windows Server (custom image)           │
│ ├─ Hyper-V role                         │
│ ├─ SLB MUX service                      │
│ ├─ BGP daemon                           │
│ ├─ VxLAN/NVGRE engine                   │
│ └─ Telemetry agents                     │
└────────────────────────────────────────┘
```

### Compute Host Physical Server

```
Typical Azure Compute Server:
┌────────────────────────────────────────┐
│ Dell/HP/Lenovo Server                   │
│                                         │
│ Hardware:                               │
│ ├─ CPUs: 2x Intel Xeon (20-40 cores)    │
│ ├─ RAM: 256-512 GB                      │
│ ├─ Storage: Local SSDs + Remote         │
│ ├─ Network: 2x 25Gbps NICs (bonded)     │
│ │  └─ SR-IOV capable                    │
│ └─ Power: Dual PSU                      │
│                                         │
│ Software:                               │
│ ├─ Windows Server Hyper-V               │
│ ├─ SLB Host Agent service               │
│ ├─ Virtual Switch (vSwitch)             │
│ ├─ Azure Fabric Agent                   │
│ └─ Customer VMs (10-40 VMs per host)    │
└────────────────────────────────────────┘

Network Configuration:
┌────────────────────────────────────────┐
│ Physical NIC → vSwitch                  │
│                                         │
│ vSwitch Ports:                          │
│ ├─ Management NIC (host OS)             │
│ ├─ VM1 vNIC (your backend VM)           │
│ ├─ VM2 vNIC (another customer)          │
│ └─ VM3-40 vNICs                         │
│                                         │
│ SLB Host Agent intercepts:              │
│ - All inbound traffic to VMs            │
│ - All outbound traffic from VMs         │
│ - Enforces load balancing               │
└────────────────────────────────────────┘
```

### Network Fabric

**Physical Network Topology**:

```
Three-Tier Architecture:

Tier 1: Border (Edge):
┌────────────────────────────────────────┐
│ Border Routers                          │
│ ├─ Connect to internet                  │
│ ├─ BGP peering with ISPs                │
│ ├─ Announce Azure public IP ranges      │
│ └─ DDoS protection appliances           │
└────────────────────────────────────────┘
        ↕
Tier 2: Spine (Core):
┌────────────────────────────────────────┐
│ Spine Switches (high-capacity)          │
│ ├─ 100-400 Gbps links                   │
│ ├─ Full mesh connectivity               │
│ ├─ ECMP routing                         │
│ └─ Connects all ToR switches            │
└────────────────────────────────────────┘
        ↕
Tier 3: ToR (Access):
┌────────────────────────────────────────┐
│ Top-of-Rack Switches                    │
│ ├─ One per rack (~40 servers)           │
│ ├─ 10/25 Gbps to servers                │
│ ├─ 100 Gbps uplinks to spine            │
│ └─ VxLAN termination                    │
└────────────────────────────────────────┘
        ↕
Tier 4: Servers:
┌────────────────────────────────────────┐
│ Physical Servers (Compute Hosts)        │
│ └─ Your backend VMs run here            │
└────────────────────────────────────────┘
```

### Public IP Address Assignment

**How Your Frontend IP Gets to Azure**:

```
Public IP Creation:

1. You create Public IP in portal:
   Resource: "MyPublicIP"
   IP: 20.50.60.10
   SKU: Standard

2. Azure assigns from regional pool:
   ┌────────────────────────────────────┐
   │ Azure Public IP Pool               │
   │ West Europe: 20.50.60.0/22         │
   │ ├─ Available: 20.50.60.10 ← Yours │
   │ ├─ In-use: ...                     │
   │ └─ Reserved: ...                   │
   └────────────────────────────────────┘

3. IP registered with Internet Routing:
   - Azure announces IP block to internet
   - BGP updates propagate globally
   - Internet routers learn: 20.50.60.0/22 → Microsoft

4. You attach IP to Load Balancer Frontend
   
5. Azure Internal Routing:
   - IP → Load Balancer ID
   - Load Balancer → MUX pool
   - MUXes announce IP internally (BGP)
   - Border routers route 20.50.60.10 → MUXes

Internet Routing Table:
┌────────────────────────────────────────┐
│ Destination: 20.50.60.0/22             │
│ AS Path: [ISP AS] [Microsoft AS]       │
│ Next Hop: Microsoft Edge Router         │
│                                         │
│ Anyone on internet can reach this IP    │
└────────────────────────────────────────┘
```

### Load Balancer Control Plane Storage

```
Configuration Storage Architecture:

┌────────────────────────────────────────┐
│ Azure Resource Manager (ARM)            │
│ - REST API frontend                     │
│ - Authentication/Authorization          │
│ - Request validation                    │
└────────────────────────────────────────┘
          ↓
┌────────────────────────────────────────┐
│ Resource Provider (RP) Layer            │
│ Microsoft.Network/loadBalancers         │
│ - Business logic                        │
│ - Resource lifecycle management         │
└────────────────────────────────────────┘
          ↓
┌────────────────────────────────────────┐
│ Azure SQL Database (geo-replicated)     │
│                                         │
│ Tables:                                 │
│ ├─ LoadBalancers                        │
│ │  ├─ ID, Name, Location, SKU          │
│ │  └─ FrontendConfigurations           │
│ ├─ BackendPools                         │
│ │  └─ Member IPs/NICs                  │
│ ├─ LoadBalancingRules                   │
│ ├─ HealthProbes                         │
│ └─ InboundNatRules                      │
│                                         │
│ Replication:                            │
│ - Primary: West Europe                  │
│ - Secondary: North Europe               │
│ - Automatic failover                    │
└────────────────────────────────────────┘
          ↓
┌────────────────────────────────────────┐
│ SLB Controller VMs                      │
│ - Read configuration from database      │
│ - Compile into data plane policies      │
│ - Push to MUXes and Host Agents         │
│ - Monitor health probes                 │
└────────────────────────────────────────┘
          ↓
┌────────────────────────────────────────┐
│ Data Plane (MUXes + Host Agents)        │
│ - Receive policy updates                │
│ - Apply configuration                   │
│ - Process packets                       │
└────────────────────────────────────────┘

Configuration Change Timeline:
T+0s: You update load balancing rule
T+0.5s: ARM validates and saves to DB
T+1s: DB replication completes
T+2s: SLB Controller reads change
T+3s: Controller compiles new policy
T+5s: Policy pushed to affected MUXes/Hosts
T+10s: All data plane updated
T+15s: Portal shows "Succeeded"
```

### Monitoring and Telemetry

```
Telemetry Collection Architecture:

┌────────────────────────────────────────┐
│ Data Sources:                           │
│ ├─ MUX VMs                              │
│ │  ├─ Packets processed/sec             │
│ │  ├─ Backend selections                │
│ │  └─ Errors                            │
│ ├─ Host Agents                          │
│ │  ├─ Flow table size                   │
│ │  ├─ SNAT port usage                   │
│ │  └─ Connection counts                 │
│ └─ Health Probe Agents                  │
│    ├─ Probe success/failure             │
│    └─ Response times                    │
└────────────────────────────────────────┘
          ↓
┌────────────────────────────────────────┐
│ Azure Monitor (Telemetry Pipeline)      │
│ ├─ Metrics ingestion (1-min intervals)  │
│ ├─ Logs ingestion (near real-time)      │
│ └─ Aggregation and storage              │
└────────────────────────────────────────┘
          ↓
┌────────────────────────────────────────┐
│ Metrics Available to You:               │
│ ├─ Data path availability (SLA)         │
│ ├─ Health probe status                  │
│ ├─ SNAT connection count                │
│ ├─ SNAT port allocation                 │
│ ├─ Byte/packet count                    │
│ └─ SYN packets                          │
└────────────────────────────────────────┘

Example Metric Query:
"Show me SNAT port usage for last hour"
→ Query Azure Monitor
→ Returns: 45% ports used (healthy)
```

-----

## Layer 8: Practical Examples

### Example 1: Basic Web Application

**Scenario**: Load balance HTTP traffic across 3 web servers

```
Requirements:
- Public-facing web application
- 3 backend VMs (web servers)
- HTTP traffic on port 80
- Health checks on /health endpoint
- Distribute traffic evenly
```

**Architecture**:

```
Internet (users)
    ↓
Public IP: 40.50.60.70
    ↓
Load Balancer: "WebLB"
  ├─ Frontend: 40.50.60.70:80
  ├─ Backend Pool: "WebServers"
  │  ├─ VM1: 10.0.1.4 (nginx)
  │  ├─ VM2: 10.0.1.5 (nginx)
  │  └─ VM3: 10.0.1.6 (nginx)
  ├─ Health Probe: HTTP /health
  └─ Rule: Port 80 → Port 80
```

**Configuration**:

```json
{
  "name": "WebLB",
  "sku": {"name": "Standard"},
  "properties": {
    "frontendIPConfigurations": [
      {
        "name": "WebFrontend",
        "properties": {
          "publicIPAddress": {
            "id": "/subscriptions/.../publicIPAddresses/WebPublicIP"
          }
        }
      }
    ],
    "backendAddressPools": [
      {
        "name": "WebServers",
        "properties": {
          "loadBalancerBackendAddresses": [
            {"properties": {"ipAddress": "10.0.1.4"}},
            {"properties": {"ipAddress": "10.0.1.5"}},
            {"properties": {"ipAddress": "10.0.1.6"}}
          ]
        }
      }
    ],
    "probes": [
      {
        "name": "HTTPHealthProbe",
        "properties": {
          "protocol": "Http",
          "port": 80,
          "requestPath": "/health",
          "intervalInSeconds": 15,
          "numberOfProbes": 2
        }
      }
    ],
    "loadBalancingRules": [
      {
        "name": "HTTPRule",
        "properties": {
          "frontendIPConfiguration": {
            "id": ".../frontendIPConfigurations/WebFrontend"
          },
          "backendAddressPool": {
            "id": ".../backendAddressPools/WebServers"
          },
          "probe": {
            "id": ".../probes/HTTPHealthProbe"
          },
          "protocol": "Tcp",
          "frontendPort": 80,
          "backendPort": 80,
          "loadDistribution": "Default"
        }
      }
    ]
  }
}
```

**Traffic Flow**:

```
User Request 1:
Client: 1.2.3.4 → LB: 40.50.60.70:80
Hash: 12345 % 3 = 0
→ Forwarded to VM1 (10.0.1.4)

User Request 2 (same user, new connection):
Client: 1.2.3.4 → LB: 40.50.60.70:80
Hash: 67890 % 3 = 0
→ Forwarded to VM1 (10.0.1.4)
(Same backend, but could be different)

User Request 3 (different user):
Client: 5.6.7.8 → LB: 40.50.60.70:80
Hash: 98765 % 3 = 2
→ Forwarded to VM3 (10.0.1.6)

Health Monitoring:
Every 15 seconds: Probe → VM1:80/health → 200 OK ✓
Every 15 seconds: Probe → VM2:80/health → 200 OK ✓
Every 15 seconds: Probe → VM3:80/health → Timeout ✗
After 2 failures: VM3 removed from pool
Distribution now: 50% VM1, 50% VM2
```

### Example 2: Three-Tier Application with Internal LB

**Scenario**: Web tier → App tier → Database tier

```
Architecture:
┌─────────────────────────────────────────┐
│ Internet                                 │
└─────────────────────────────────────────┘
              ↓
┌─────────────────────────────────────────┐
│ Public Load Balancer                     │
│ Frontend: 40.50.60.70:443                │
│ Backend: Web Tier (10.0.1.0/24)          │
└─────────────────────────────────────────┘
              ↓
┌─────────────────────────────────────────┐
│ Web Tier Subnet (10.0.1.0/24)            │
│ ├─ WebVM1: 10.0.1.4                      │
│ ├─ WebVM2: 10.0.1.5                      │
│ └─ WebVM3: 10.0.1.6                      │
└─────────────────────────────────────────┘
              ↓
┌─────────────────────────────────────────┐
│ Internal Load Balancer (App Tier)        │
│ Frontend: 10.0.2.100:8080                │
│ Backend: App Tier (10.0.2.0/24)          │
└─────────────────────────────────────────┘
              ↓
┌─────────────────────────────────────────┐
│ App Tier Subnet (10.0.2.0/24)            │
│ ├─ AppVM1: 10.0.2.4                      │
│ ├─ AppVM2: 10.0.2.5                      │
│ └─ AppVM3: 10.0.2.6                      │
└─────────────────────────────────────────┘
              ↓
┌─────────────────────────────────────────┐
│ Internal Load Balancer (Database)        │
│ Frontend: 10.0.3.100:1433                │
│ Backend: DB Tier (10.0.3.0/24)           │
└─────────────────────────────────────────┘
              ↓
┌─────────────────────────────────────────┐
│ Database Tier Subnet (10.0.3.0/24)       │
│ ├─ DBVM1: 10.0.3.4 (Primary)             │
│ └─ DBVM2: 10.0.3.5 (Secondary)           │
└─────────────────────────────────────────┘
```

**Load Balancer Configurations**:

**Public LB (Web Tier)**:

```json
{
  "name": "PublicWebLB",
  "sku": {"name": "Standard"},
  "properties": {
    "frontendIPConfigurations": [{
      "name": "PublicFrontend",
      "properties": {
        "publicIPAddress": {"id": ".../WebPublicIP"}
      }
    }],
    "backendAddressPools": [{
      "name": "WebServers",
      "properties": {
        "backendIPConfigurations": [
          {" id": ".../WebVM1/ipConfigurations/ipconfig1"},
          {"id": ".../WebVM2/ipConfigurations/ipconfig1"},
          {"id": ".../WebVM3/ipConfigurations/ipconfig1"}
        ]
      }
    }],
    "probes": [{
      "name": "HTTPSProbe",
      "properties": {
        "protocol": "Https",
        "port": 443,
        "requestPath": "/health",
        "intervalInSeconds": 15,
        "numberOfProbes": 2
      }
    }],
    "loadBalancingRules": [{
      "name": "HTTPSRule",
      "properties": {
        "frontendIPConfiguration": {"id": ".../PublicFrontend"},
        "backendAddressPool": {"id": ".../WebServers"},
        "probe": {"id": ".../HTTPSProbe"},
        "protocol": "Tcp",
        "frontendPort": 443,
        "backendPort": 443,
        "loadDistribution": "Default"
      }
    }]
  }
}
```

**Internal LB (App Tier)**:

```json
{
  "name": "InternalAppLB",
  "sku": {"name": "Standard"},
  "properties": {
    "frontendIPConfigurations": [{
      "name": "AppFrontend",
      "properties": {
        "privateIPAddress": "10.0.2.100",
        "privateIPAllocationMethod": "Static",
        "subnet": {"id": ".../subnets/AppSubnet"}
      }
    }],
    "backendAddressPools": [{
      "name": "AppServers",
      "properties": {
        "backendIPConfigurations": [
          {"id": ".../AppVM1/ipConfigurations/ipconfig1"},
          {"id": ".../AppVM2/ipConfigurations/ipconfig1"},
          {"id": ".../AppVM3/ipConfigurations/ipconfig1"}
        ]
      }
    }],
    "probes": [{
      "name": "TCPProbe",
      "properties": {
        "protocol": "Tcp",
        "port": 8080,
        "intervalInSeconds": 15,
        "numberOfProbes": 2
      }
    }],
    "loadBalancingRules": [{
      "name": "AppRule",
      "properties": {
        "frontendIPConfiguration": {"id": ".../AppFrontend"},
        "backendAddressPool": {"id": ".../AppServers"},
        "probe": {"id": ".../TCPProbe"},
        "protocol": "Tcp",
        "frontendPort": 8080,
        "backendPort": 8080,
        "loadDistribution": "SourceIP"  // Session persistence
      }
    }]
  }
}
```

**Internal LB (Database)**:

```json
{
  "name": "InternalDBLB",
  "sku": {"name": "Standard"},
  "properties": {
    "frontendIPConfigurations": [{
      "name": "DBFrontend",
      "properties": {
        "privateIPAddress": "10.0.3.100",
        "privateIPAllocationMethod": "Static",
        "subnet": {"id": ".../subnets/DBSubnet"}
      }
    }],
    "backendAddressPools": [{
      "name": "DBServers",
      "properties": {
        "backendIPConfigurations": [
          {"id": ".../DBVM1/ipConfigurations/ipconfig1"},
          {"id": ".../DBVM2/ipConfigurations/ipconfig1"}
        ]
      }
    }],
    "probes": [{
      "name": "DBProbe",
      "properties": {
        "protocol": "Tcp",
        "port": 1433,
        "intervalInSeconds": 5,
        "numberOfProbes": 2
      }
    }],
    "loadBalancingRules": [{
      "name": "DBRule",
      "properties": {
        "frontendIPConfiguration": {"id": ".../DBFrontend"},
        "backendAddressPool": {"id": ".../DBServers"},
        "probe": {"id": ".../DBProbe"},
        "protocol": "Tcp",
        "frontendPort": 1433,
        "backendPort": 1433,
        "loadDistribution": "SourceIP",  // Important for DB!
        "enableFloatingIP": true  // Required for SQL AlwaysOn
      }
    }]
  }
}
```

**Complete Request Flow**:

```
Step 1: User → Web Tier
User: 1.2.3.4 → Public LB: 40.50.60.70:443
  ↓ (Load balanced)
Web VM1: 10.0.1.4 (receives request)

Step 2: Web Tier → App Tier
Web VM1: 10.0.1.4 → Internal LB: 10.0.2.100:8080
  ↓ (Load balanced, session persistent)
App VM2: 10.0.2.5 (receives request)

Step 3: App Tier → Database
App VM2: 10.0.2.5 → Internal LB: 10.0.3.100:1433
  ↓ (Load balanced to primary)
DB VM1: 10.0.3.4 (receives query)

Step 4: Database → App Tier (Response)
DB VM1: 10.0.3.4 → App VM2: 10.0.2.5
  ↑ (Direct return via stateful tracking)

Step 5: App Tier → Web Tier (Response)
App VM2: 10.0.2.5 → Web VM1: 10.0.1.4
  ↑ (Direct return via stateful tracking)

Step 6: Web Tier → User (Response)
Web VM1: 10.0.1.4 → Public LB → User: 1.2.3.4
  ↑ (Return via load balancer)

Key Points:
✓ Three load balancers (1 public, 2 internal)
✓ Each tier independently scalable
✓ Session persistence at app tier (stateful apps)
✓ Database uses Floating IP (SQL AlwaysOn)
✓ Defense in depth (multiple LB layers)
```

### Example 3: Outbound Internet Access with NAT

**Scenario**: VMs with no public IPs need internet access

```
Problem:
- Backend VMs have only private IPs
- Need to download updates from internet
- No public IPs assigned to VMs
- Standard SKU LB (no automatic outbound)

Solution:
Use Load Balancer with outbound rule
```

**Architecture**:

```
┌─────────────────────────────────────────┐
│ Backend VMs (Private IPs only)           │
│ ├─ VM1: 10.0.1.4                         │
│ ├─ VM2: 10.0.1.5                         │
│ └─ VM3: 10.0.1.6                         │
└─────────────────────────────────────────┘
              ↓ (Outbound traffic)
┌─────────────────────────────────────────┐
│ Load Balancer                            │
│ ├─ Frontend: 40.50.60.70 (Public IP)     │
│ └─ Outbound Rule: SNAT                   │
└─────────────────────────────────────────┘
              ↓
┌─────────────────────────────────────────┐
│ Internet                                 │
│ (Updates, APIs, external services)       │
└─────────────────────────────────────────┘
```

**Configuration**:

```json
{
  "name": "OutboundLB",
  "sku": {"name": "Standard"},
  "properties": {
    "frontendIPConfigurations": [{
      "name": "OutboundFrontend",
      "properties": {
        "publicIPAddress": {"id": ".../OutboundPublicIP"}
      }
    }],
    "backendAddressPools": [{
      "name": "VMPool",
      "properties": {
        "backendIPConfigurations": [
          {"id": ".../VM1/ipConfigurations/ipconfig1"},
          {"id": ".../VM2/ipConfigurations/ipconfig1"},
          {"id": ".../VM3/ipConfigurations/ipconfig1"}
        ]
      }
    }],
    "outboundRules": [{
      "name": "OutboundRule",
      "properties": {
        "frontendIPConfigurations": [
          {"id": ".../OutboundFrontend"}
        ],
        "backendAddressPool": {
          "id": ".../VMPool"
        },
        "protocol": "All",
        "allocatedOutboundPorts": 2048,
        "idleTimeoutInMinutes": 4,
        "enableTcpReset": true
      }
    }]
  }
}
```

**How It Works**:

```
Outbound Connection:

VM1 wants to reach: api.example.com (93.184.216.34)

Step 1: VM1 initiates connection
Packet from VM:
Source: 10.0.1.4:55000
Dest: 93.184.216.34:443

Step 2: Load Balancer SNAT
LB applies Source NAT:
Source: 40.50.60.70:10000  ← Changed!
Dest: 93.184.216.34:443

LB creates SNAT mapping:
{
  internal: 10.0.1.4:55000,
  external: 40.50.60.70:10000,
  dest: 93.184.216.34:443,
  allocated_port: 10000
}

Step 3: Packet sent to internet
Source: 40.50.60.70:10000
Dest: 93.184.216.34:443

Step 4: Return traffic
Response from: 93.184.216.34:443
To: 40.50.60.70:10000

Step 5: LB reverse SNAT
LB looks up port 10000:
→ Internal: 10.0.1.4:55000
Changes destination:
Source: 93.184.216.34:443
Dest: 10.0.1.4:55000

Step 6: Packet delivered to VM1
VM1 receives response
```

**SNAT Port Allocation**:

```
Port Pool Management:

Frontend IP: 40.50.60.70
Available ports: 1024-65535 = 64,512 ports
(ports 0-1023 are reserved)

Backend Pool: 3 VMs
Allocated per VM: 2,048 ports
(configured in outboundRule)

VM1 Port Range: 10,000 - 12,047
VM2 Port Range: 12,048 - 14,095
VM3 Port Range: 14,096 - 16,143

Each VM can have up to 2,048 concurrent outbound connections

Port Exhaustion:
If VM1 uses all 2,048 ports:
→ New outbound connections fail
→ Application errors
→ Monitor metric: "SNAT Connection Count"

Solutions:
1. Increase allocatedOutboundPorts
2. Add more frontend IPs
3. Use NAT Gateway instead (better)
```

**Better Alternative: NAT Gateway**:

```
Instead of Outbound Rule, use NAT Gateway:

Advantages:
✓ 64,000 ports per IP
✓ Can use multiple IPs (scale to millions of connections)
✓ More reliable for high-volume outbound
✓ Dedicated service (not shared with inbound)
✓ Easier to manage

Configuration:
┌─────────────────────────────────────────┐
│ NAT Gateway                              │
│ ├─ Public IP: 40.50.60.80                │
│ ├─ Attached to Subnet: 10.0.1.0/24       │
│ └─ All outbound traffic uses NAT Gateway │
└─────────────────────────────────────────┘

No outbound rule needed on Load Balancer!
```

### Example 4: High Availability with Availability Zones

**Scenario**: Zone-redundant application

```
Requirements:
- Deploy across 3 availability zones
- Survive zone failure
- 99.99% SLA
- Load balance across all zones
```

**Architecture**:

```
Azure Region: West Europe

Zone-Redundant Frontend:
Public IP: Zone-redundant
Load Balancer: Zone-redundant
  ↓
┌─────────────────────────────────────────┐
│ Availability Zone 1                      │
│ ├─ VM1: 10.0.1.4                         │
│ └─ VM2: 10.0.1.5                         │
└─────────────────────────────────────────┘

┌─────────────────────────────────────────┐
│ Availability Zone 2                      │
│ ├─ VM3: 10.0.1.6                         │
│ └─ VM4: 10.0.1.7                         │
└─────────────────────────────────────────┘

┌─────────────────────────────────────────┐
│ Availability Zone 3                      │
│ ├─ VM5: 10.0.1.8                         │
│ └─ VM6: 10.0.1.9                         │
└─────────────────────────────────────────┘
```

**Configuration**:

```json
{
  "name": "ZoneRedundantLB",
  "sku": {
    "name": "Standard",
    "tier": "Regional"
  },
  "properties": {
    "frontendIPConfigurations": [{
      "name": "ZoneFrontend",
      "properties": {
        "publicIPAddress": {
          "id": ".../ZoneRedundantPublicIP"
        }
      },
      "zones": []  // Empty = zone-redundant
    }],
    "backendAddressPools": [{
      "name": "ZoneBackends",
      "properties": {
        "loadBalancerBackendAddresses": [
          {
            "properties": {
              "ipAddress": "10.0.1.4",
              "virtualNetwork": {"id": ".../VNet"}
            }
          },
          // ... other VMs
        ]
      }
    }],
    "probes": [{
      "name": "HealthProbe",
      "properties": {
        "protocol": "Tcp",
        "port": 80,
        "intervalInSeconds": 5,
        "numberOfProbes": 2
      }
    }],
    "loadBalancingRules": [{
      "name": "HTTPRule",
      "properties": {
        "frontendIPConfiguration": {"id": ".../ZoneFrontend"},
        "backendAddressPool": {"id": ".../ZoneBackends"},
        "probe": {"id": ".../HealthProbe"},
        "protocol": "Tcp",
        "frontendPort": 80,
        "backendPort": 80
      }
    }]
  }
}
```

**Failure Scenario**:

```
Normal Operation:
├─ Zone 1: VM1, VM2 (Healthy) ✓
├─ Zone 2: VM3, VM4 (Healthy) ✓
└─ Zone 3: VM5, VM6 (Healthy) ✓

Traffic distributed: ~33% per zone

Zone 1 Failure (e.g., datacenter power issue):
├─ Zone 1: VM1, VM2 (Unhealthy) ✗
├─ Zone 2: VM3, VM4 (Healthy) ✓
└─ Zone 3: VM5, VM6 (Healthy) ✓

What Happens:
1. Health probes to Zone 1 fail
2. After 2 consecutive failures (~10 seconds):
   - VM1, VM2 marked unhealthy
3. Load balancer stops sending traffic to Zone 1
4. Traffic redistributed:
   - Zone 2: 50% of traffic
   - Zone 3: 50% of traffic
5. Application remains available!
6. Capacity reduced by 33%

Automatic Recovery:
When Zone 1 recovers:
1. Health probes succeed
2. VM1, VM2 marked healthy
3. Traffic automatically resumes to Zone 1
4. Load redistributed to all 3 zones

SLA Impact:
Zone-redundant: 99.99% uptime
Single zone: 99.95% uptime
(Zone failure = no outage for zone-redundant)
```

-----

## Summary: The Complete Picture

### What You Now Understand About Azure Load Balancer

**Layer 1 (Portal)**:

- Simple UI for creating load balancers
- Abstracts complex distributed system
- Configuration stored and replicated

**Layer 2 (Components)**:

- Frontend IPs (public/internal)
- Backend pools (VMs, IPs, VMSS)
- Health probes (TCP, HTTP, HTTPS)
- Load balancing rules (port mapping)
- NAT rules (direct access)
- Outbound rules (SNAT)

**Layer 3 (How It Works)**:

- 5-tuple hash distribution
- Stateful connection tracking
- Health-based routing
- Session persistence options
- Zone redundancy

**Layer 4 (Technology)**:

- Software Load Balancer (SLB)
- Multiplexers (MUXes)
- Host Agents
- VxLAN encapsulation
- ECMP routing

**Layer 5 (TCP/IP)**:

- Operates at Layer 3 (Transport) and Layer 2 (IP)
- SNAT/DNAT for address translation
- Port mapping and translation
- Does NOT inspect application layer

**Layer 6 (Packet Flow)**:

- Client → Border Router → MUX → Host Agent → VM
- VM → Host Agent → MUX → Border Router → Client
- Connection state tracking throughout

**Layer 7 (Physical)**:

- MUXes on specialized compute hosts
- Distributed across availability zones
- VxLAN encapsulation in datacenter
- No single point of failure

### Key Insights

**Azure Load Balancer IS**:
✅ Layer 4 (Transport) load balancer
✅ Fully managed, highly available
✅ Software-defined, distributed system
✅ Protocol-agnostic (TCP/UDP)
✅ Sub-millisecond latency
✅ Scales to millions of flows
✅ Zone-redundant capable
✅ Integrated with Azure SDN

**Azure Load Balancer is NOT**:
❌ Layer 7 load balancer (use App Gateway)
❌ Web Application Firewall (use App Gateway)
❌ Content-aware routing (use App Gateway)
❌ SSL termination (use App Gateway)
❌ HTTP-specific (it’s protocol-agnostic)

### Comparison with Other Azure Services

```
Use Azure Load Balancer when:
✓ Need Layer 4 load balancing
✓ Any TCP/UDP protocol
✓ Internal or public traffic
✓ Simple distribution
✓ Cost-effective solution

Use Application Gateway when:
✓ HTTP/HTTPS only
✓ URL-based routing needed
✓ SSL termination needed
✓ WAF needed
✓ Cookie-based session persistence

Use Traffic Manager when:
✓ DNS-based global routing
✓ Multi-region failover
✓ Geographic routing

Use Front Door when:
✓ Global HTTP load balancing
✓ Edge caching (CDN)
✓ WAF at edge
✓ SSL offload at edge
```

### Best Practices Summary

**1. SKU Selection**:

- Use Standard SKU (not Basic)
- More features, better SLA
- Required for production

**2. Health Probes**:

- Use HTTP/HTTPS probes for web apps
- Create dedicated health endpoints
- Check actual application health
- Set appropriate intervals (5-15 seconds)

**3. Distribution**:

- Default (5-tuple): Stateless apps
- SourceIP: Stateful apps needing persistence
- Consider app architecture when choosing

**4. Outbound Connectivity**:

- Use NAT Gateway (not outbound rules)
- Monitor SNAT port usage
- Plan for port exhaustion

**5. High Availability**:

- Deploy across availability zones
- Zone-redundant frontend IPs
- Multiple backends per zone
- Monitor health probe status

**6. Security**:

- Standard SKU: Closed by default (good!)
- Use NSGs with load balancer
- Allow health probe IP: 168.63.129.16
- Separate inbound/outbound frontend IPs

**7. Monitoring**:

- Watch metrics:
  - Data path availability
  - Health probe status
  - SNAT port usage
  - Packet/byte count
- Set up alerts for probe failures

### Connection to Broader Networking

**Load Balancer uses fundamentals you learned**:

- TCP/IP layers → Where LB operates
- IP addressing → VIP/DIP translation
- Subnetting → Internal LB placement
- NAT → SNAT/DNAT concepts
- Routing → How traffic reaches backends
- Health checks → TCP/HTTP protocols

**Every Azure load balancing service builds on these concepts**:

- Basic Load Balancer → Foundation
- Standard Load Balancer → Enhanced features
- Application Gateway → Layer 7 on top of Layer 4
- Traffic Manager → DNS routing
- Front Door → Global Layer 7

-----

## Next Steps for Your Learning

Now that you understand Azure Load Balancer deeply, great next topics:

1. **Azure Application Gateway**
- Layer 7 load balancing
- URL-based routing
- WAF integration
- SSL termination
1. **Azure Virtual Network**
- Foundation for all networking
- Subnets and address spaces
- VNet peering
- Routing tables
1. **Azure VPN Gateway**
- Hybrid connectivity
- Site-to-Site VPN
- Point-to-Site VPN
- ExpressRoute
1. **Azure Private Link**
- Private connectivity to PaaS
- Service endpoints
- Private endpoints
1. **Azure Firewall**
- Network security appliance
- DNAT, SNAT rules
- Threat intelligence
- Integration with LB

Each one builds on load balancing concepts!

This document follows the same 8-layer pattern to help you understand Azure Load Balancer from the Azure Portal interface down to the physical implementation and network packets. Understanding how load balancing works at this level will help you design, troubleshoot, and optimize Azure architectures effectively.
