Small Business Network Design, Implementation, and Security HardeningThis repository contains the configuration documentation, architecture designs, and hardening protocols for deploying a secure, virtualized small business network infrastructure. The practical environment is simulated via VMware Workstation, utilizing OpenWRT as the primary enterprise edge gateway/firewall to protect an internal IT Consultancy firm based in Brisbane.🏛 Network Architecture & TopologyThe infrastructure establishes a strict boundary between the external public network (WAN) and internal company assets (LAN). The network services an estimated staff of eight users, hosting local web repositories and enforcing baseline security controls.Network Address Allocation MatrixDevice / Host RoleHardware InterfaceAllocated IP AddressFunctional ScopeOpenWRT Gatewayeth0 (WAN)DHCP (Dynamic)Upstream Public Internet AccessOpenWRT Gatewayeth1 (LAN)5.6.7.8/24Default Corporate GatewayWindows WorkstationVirtual Ethernet5.6.7.9/24Staff Endpoint / Auditing HostInternal Web ServerLoopback / /wwwShared Gateway IPCorporate Landing Page Hosting🛠 Deployment & Setup InstructionsPhase 1: Virtual Hypervisor Environment SetupImport the OpenWRT image into VMware Workstation.Provision two network adapters for the OpenWRT Virtual Machine:Adapter 1 (WAN): Map to Bridged or NAT mode to receive automated DHCP routing.Adapter 2 (LAN): Map to a custom Host-Only segment (VMnet) to build the isolated internal network.Configure the Windows Host network interface properties on the same Host-Only VMnet segment with the manual configuration:IP Address: 5.6.7.9Subnet Mask: 255.255.255.0Gateway: 5.6.7.8Phase 2: Interface Mapping via OpenWRT ShellAccess the OpenWRT command-line interface directly via the hypervisor console and edit the system network layout:Bash# Open the configuration document
vi /etc/config/network

# Update or append the lan interface configuration block
config interface 'lan'
    option device 'eth1'
    option proto 'static'
    option ipaddr '5.6.7.8'
    option netmask '255.255.255.0'

# Commit changes and recycle the network service daemons
service network restart
🔒 Security Hardening Protocols Implemented1. Root Credential InitializationEliminate zero-day brute-force vectors by dropping the default blank credentials:Bashpasswd root
2. Disabling Insecure Password Authentication Over SSHEnforce cryptographic verification by shifting authentication mechanics from basic password strings to public/private key-pairs.Generate an asymmetric key-pair on the Windows Client Host via Git Bash:Bashssh-keygen -t rsa -b 4096 -C "student@consultancy.local"
Upload the public identity block (id_rsa.pub) to OpenWRT via the LuCI System Web GUI under System -> Administration -> SSH-Keys.Access the OpenWRT SSH configuration parameters, disable standard entry pathways, and restart the service daemon:Bash   # Modify dropbear configurations to drop password challenges
   uci set dropbear.@dropbear[0].PasswordAuth='0'
   uci set dropbear.@dropbear[0].RootPasswordAuth='0'
   uci commit dropbear
   /etc/init.d/dropbear restart
3. Attack Surface Reduction (IPv6 Elimination)Because the environment is deployed inside a dedicated IPv4 scope, the unnecessary IPv6 processing service (odhcpd) was stopped and permanently disabled from system initialization paths:Bash/etc/init.d/odhcpd stop
/etc/init.d/odhcpd disable
🌐 Corporate Web Server DeploymentA light corporate showcase webpage was successfully implemented locally inside the edge infrastructure directory.Pivot to the local web application folder: cd /wwwCreate and design a customized, structured corporate profile using a static file deployment format (student.html).Fire up an endpoint browser on the Windows Host machine and verify server functionality by executing a tracking call directly to:HTTPhttp://5.6.7.8/student.html
🧱 Enterprise Firewall Ruleset Reference (/etc/config/firewall)The following strict firewall filtering profiles were successfully introduced via the internal OpenWRT systems to enforce structural data access restrictions.Rule 1: HTTP Traffic Block / Drop RulePlaintextName:           Block-HTTP-Traffic
Source Zone:    lan (Internal Workstations)
Destination:    Device (Local Web Server)
Protocol:       TCP
Port:           80
Action:         REJECT / DROP
Rule 2: SSH Port Relocation & Security RulePlaintextName:           Relocate-SSH-Management
New Port:       2222 (Configured under dropbear daemon configuration files)
Protocol:       TCP
Action:         ACCEPT
Verification:   ssh root@5.6.7.8 -p 2222
Rule 3: Network Stealth Mode (ICMP Ping Drop)PlaintextName:           Block-ICMP-Inbound
Source Zone:    lan
Destination:    Device
Protocol:       ICMP (ping)
Action:         DROP
Rule 4: Custom Alternate Port Management RestrictionPlaintextName:           Restrict-Port-81-Access
Source Zone:    lan
Destination:    Device (LuCI Alternate Port Interface)
Protocol:       TCP
Port:           81
Action:         REJECT
📈 Network Traffic Analysis LogsDeep-packet payload validation testing was carried out through Wireshark tracking captures to monitor security performance:HTTP Cleartext Vulnerability Capture: Wire captures confirm that standard HTTP (TCP Port 80) interactions expose application queries in cleartext. Transitioning critical areas to TLS v1.2 / TLS v1.3 remains a baseline priority to keep core transaction payloads secure from malicious sniffing efforts.SSH v2 Transport Layer Verification: Captured sequences reveal successful negotiations utilizing strong cryptographic protocols (AES-128-CTR / ChaCha20). Utilizing explicit SSH v2 specifications protects the company's admin controls against downgraded cryptanalysis exploits or man-in-the-middle data manipulation.