A lightweight Virtual Private Cloud (VPC) implementation on Linux using network namespaces, bridges, and iptables. Build isolated networks with routing, NAT, and firewall rules - just like AWS VPC!
🎯 Features

✅ VPC Creation - Isolated virtual networks with custom CIDR blocks
✅ Multiple Subnets - Public and private subnet support
✅ Inter-Subnet Routing - Communication between subnets via bridge
✅ NAT Gateway - Internet access for public subnets
✅ DNS Configuration - Automatic DNS setup for namespaces
✅ Network Isolation - Complete VPC-level isolation
✅ Simple CLI - Easy-to-use command-line interface

🚀 Quick Start
Prerequisites
bash# Required tools
sudo apt-get install -y iproute2 iptables jq

# Verify installation
which ip iptables jq
Installation
bash# Clone the repository
git clone https://github.com/Mukky001/HNG_Task4_VPC_Setup
cd vpc-network-simulator

# Make executable
chmod +x vpcctl

# Optional: Install system-wide
sudo ln -s $(pwd)/vpcctl /usr/local/bin/vpcctl
📖 Usage
Create a VPC
bashsudo ./vpcctl create --name dev-vpc --cidr 10.0.0.0/16
Add Subnets
bash# Public subnet (with internet access)
sudo ./vpcctl add-subnet --vpc dev-vpc --name public --cidr 10.0.1.0/24 --type public

# Private subnet (internal only)
sudo ./vpcctl add-subnet --vpc dev-vpc --name private --cidr 10.0.2.0/24 --type private
Enable Internet Access (NAT)
bashsudo ./vpcctl enable-nat --vpc dev-vpc --subnet public
List VPCs
bashsudo ./vpcctl list
Delete VPC
bashsudo ./vpcctl delete --name dev-vpc
🧪 Testing Connectivity
Test Gateway Connection
bash# Ping the gateway from public subnet
sudo ip netns exec ns-dev-vpc-public ping -c 3 10.0.1.1
Test Inter-Subnet Communication
bash# From public subnet, ping private subnet
sudo ip netns exec ns-dev-vpc-public ping -c 3 10.0.2.10
Test Internet Access
bash# Ping Google DNS
sudo ip netns exec ns-dev-vpc-public ping -c 3 8.8.8.8

# Test HTTPS
sudo ip netns exec ns-dev-vpc-public curl -I https://www.google.com
Deploy Web Server
bash# Start web server in private subnet
sudo ip netns exec ns-dev-vpc-private python3 -m http.server 8080 &

# Access from public subnet
sudo ip netns exec ns-dev-vpc-public curl http://10.0.2.10:8080
```

## 🏗️ Architecture
```
┌─────────────────────────────────────────────────────────┐
│                    HOST MACHINE                          │
│                                                          │
│  ┌────────────── VPC: dev-vpc (10.0.0.0/16) ──────────┐ │
│  │                                                      │ │
│  │         [Bridge: br-dev-vpc]                        │ │
│  │              (VPC Router)                           │ │
│  │                     │                                │ │
│  │         ┌───────────┴───────────┐                   │ │
│  │         │                       │                   │ │
│  │    [Gateway: 10.0.1.1]    [Gateway: 10.0.2.1]      │ │
│  │         │                       │                   │ │
│  │    [veth pair]            [veth pair]               │ │
│  │         │                       │                   │ │
│  │  ┌──────▼──────┐         ┌──────▼──────┐          │ │
│  │  │ Public NS   │         │ Private NS  │          │ │
│  │  │ 10.0.1.10   │◄────────►│ 10.0.2.10   │          │ │
│  │  │             │         │             │          │ │
│  │  │ [NAT: ON]   │         │ [NAT: OFF]  │          │ │
│  │  └─────────────┘         └─────────────┘          │ │
│  │         │                                            │ │
│  └─────────┼────────────────────────────────────────────┘ │
│            │                                             │
│            │ (NAT via iptables)                         │
│            ↓                                             │
│      [Internet] 🌐                                      │
└─────────────────────────────────────────────────────────┘
🔧 How It Works
Network Namespaces

Each subnet is a separate network namespace
Provides complete network isolation
Acts like a virtual machine's network stack

Linux Bridge

Functions as the VPC router
Forwards traffic between subnets
Connects all veth interfaces

Veth Pairs

Virtual ethernet cables
One end in namespace (subnet)
Other end attached to bridge

NAT (MASQUERADE)

Translates private IPs to host IP
Enables internet access for public subnets
Implemented with iptables rules

DNS Resolution

Configured per-namespace
Uses public DNS servers (8.8.8.8, 1.1.1.1)
Stored in /etc/netns/<namespace>/resolv.conf

📋 Commands Reference
CommandDescriptionExamplecreateCreate a new VPCvpcctl create --name vpc1 --cidr 10.0.0.0/16add-subnetAdd subnet to VPCvpcctl add-subnet --vpc vpc1 --name public --cidr 10.0.1.0/24 --type publicenable-natEnable internet accessvpcctl enable-nat --vpc vpc1 --subnet publiclistList all VPCsvpcctl listdeleteDelete a VPCvpcctl delete --name vpc1
🐛 Troubleshooting
Ping to gateway fails
bash# Check if bridge has IP
ip addr show br-dev-vpc

# Check namespace IP
sudo ip netns exec ns-dev-vpc-public ip addr show

# Check routing
sudo ip netns exec ns-dev-vpc-public ip route show
Internet access not working
bash# Check NAT rules
sudo iptables -t nat -L POSTROUTING -n -v | grep 10.0.1.0

# Check IP forwarding
cat /proc/sys/net/ipv4/ip_forward  # Should be 1

# Check DNS
sudo ip netns exec ns-dev-vpc-public cat /etc/resolv.conf
DNS resolution fails
bash# Recreate DNS config
sudo mkdir -p /etc/netns/ns-dev-vpc-public
echo "nameserver 8.8.8.8" | sudo tee /etc/netns/ns-dev-vpc-public/resolv.conf

# Test DNS
sudo ip netns exec ns-dev-vpc-public nslookup google.com
📝 State Management
VPC state is stored in /etc/vpcctl/vpcs.json:
json{
  "dev-vpc": {
    "cidr": "10.0.0.0/16",
    "bridge": "br-dev-vpc",
    "subnets": {
      "public": {
        "cidr": "10.0.1.0/24",
        "type": "public",
        "namespace": "ns-dev-vpc-public",
        "namespace_ip": "10.0.1.10/24",
        "gateway_ip": "10.0.1.1/24"
      }
    },
    "created_at": "2025-11-10T12:00:00Z"
  }
}

👤 Author
Muktar Akinola
