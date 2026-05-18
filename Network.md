Network Topology and Configuration Guide (NETWORK.md)This document serves as the structural reference for the physical, logical, and virtualized network design implemented for our small business IT consultancy scenario based in Brisbane.🗺️ 1. Logical Network DesignThe network infrastructure is built to isolate internal company endpoints from public internet interfaces while maintaining steady, managed routing pathways.1.1 Conceptual Topology Block DiagramPlaintext       [ Public Upstream Internet ]
                    │
                    ▼ [Dynamic DHCP Public IP]
           ┌─────────────────┐
           │  OpenWRT Edge   │
           │  Router/Firewall│
           └─────────────────┘
                    │
                    ▼ [Static Gateway: 5.6.7.8/24]
     ───────────────────────┬─────────────────────── (Internal Corporate LAN)
                            │
                            ▼ [Static Endpoint IP: 5.6.7.9/24]
                   ┌─────────────────┐
                   │  Windows Host   │
                   │  (Workstation)  │
                   └─────────────────┘
📋 2. Interface Configuration & Address MappingThe network schema uses a specific IPv4 classless inter-domain routing (CIDR) profile to organize internal resources cleanly.Hardware VM InterfaceRouter NameLogical ZoneAssignment TypeTarget IP Range / NetmaskNetwork Adapter 1eth0WANDHCP (Dynamic)Upstream Provider ManagedNetwork Adapter 2eth1LANStatic5.6.7.8 / 255.255.255.0Virtual Ethernet—LAN HostStatic5.6.7.9 / 255.255.255.0⚙️ 3. Low-Level Configuration Files3.1 OpenWRT Network Configuration Blueprint (/etc/config/network)The structural mapping of the local area network interface card is declared in the standard OpenWRT network layout schema below. This manual binding explicitly links the lan interface to the secondary virtual interface controller (eth1).Plaintextconfig interface 'loopback'
    option device 'lo'
    option proto 'static'
    option ipaddr '127.0.0.1'
    option netmask '255.0.0.0'

config globals 'globals'
    option ula_prefix 'fd54:ad41:9645::/48'

config interface 'wan'
    option device 'eth0'
    option proto 'dhcp'

config interface 'lan'
    option device 'eth1'
    option proto 'static'
    option ipaddr '5.6.7.8'
    option netmask '255.255.255.0'
3.2 Windows Client Network Configuration ParametersTo establish a stable connection with the gateway, the primary host Windows workstation adapter must be configured with these specific parameters within the IPv4 properties window:Static IPv4 Address Address: 5.6.7.9Subnet Mask Designation: 255.255.255.0Default Router Gateway Point: 5.6.7.8Primary Domain Name Server (DNS): 1.1.1.1 (Cloudflare Anycast Engine)🧪 4. Step-by-Step Network Provisioning ProcessStep 4.1: Initializing Hypervisor Network CardsPower down the OpenWRT Virtual Machine within VMware Workstation.Open the VM Hardware Settings pane and ensure exactly two network controllers are added:Network Adapter (WAN): Map directly to the hypervisor's NAT or Bridged option.Network Adapter 2 (LAN): Map explicitly to a dedicated Host-Only virtual switch (e.g., VMnet2 or VMnet3).Modify the properties of the Windows Client VM, ensuring its primary network card is assigned to the exact same Host-Only switch used for the router's LAN.Step 4.2: Applying the OpenWRT Network LayoutBoot up the OpenWRT instance and access the terminal console using the built-in VMware window:Bash# 1. Edit the interface parameters directly from the terminal
vi /etc/config/network

# 2. Complete adjustments to matches the blueprint in Section 3.1
# 3. Save modifications and close the editor (:wq)

# 4. Recycle network subsystems to apply settings instantly
service network restart
🔍 5. Verification and Routing DiagnosticsOnce configurations are successfully pushed across both instances, execute these baseline validation steps to confirm proper network routing:5.1 Gateway Edge Connectivity Audit (WAN Verification)From the OpenWRT terminal, issue an outbound ICMP request sequence to verify external translation and resolution:Bashping -c 4 1.1.1.1
Success Criteria: 0% packet loss profile. This proves the WAN interface (eth0) is pulling data correctly from the upstream provider and routing outbound requests.5.2 Internal Host-to-Gateway Link Audit (LAN Verification)From the Windows host workstation terminal (using Git Bash or CMD), issue an internal ping toward the newly mapped static gateway address:Bashping 5.6.7.8
Success Criteria: Stable round-trip time (RTT) responses without packet timeouts. This confirms that the VMware Host-Only segment is passing traffic securely between the endpoints.5.3 Administrative Web Panel Check (LuCI Access)Open a web browser on the Windows host machine and point the URL directly to the gateway:HTTPhttp://5.6.7.8/
Success Criteria: The OpenWRT LuCI landing dashboard loads successfully, confirming that the HTTP server daemon is listening and accessible across the local network segment.