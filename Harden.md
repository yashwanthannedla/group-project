Network Security Hardening Guide (HARDEN.md)
This document outlines the standard operating procedures used to harden the OpenWRT edge gateway and the internal virtualized network infrastructure for our small business IT consultancy scenario.

🛑 1. Operational Security Baseline
1.1 Changing Default Administrative Credentials
Out-of-the-box networking equipment frequently ships with blank or easily guessable factory credentials. To eliminate immediate brute-force access vectors, establish a robust password for the root administrative account.

Connect to the OpenWRT console or establish an initial SSH session.

Execute the password utility command:

Bash
passwd root
Enter a cryptographically secure passphrase (minimum 12 characters, mixing uppercase, lowercase, numbers, and special symbols).

1.2 Examining Cryptographic Password Storage
OpenWRT stores system user accounts and their corresponding password records securely inside the shadow password file. To verify the cryptographic strength of the system:

Bash
cat /etc/shadow | grep root
Security Audit: Ensure the hashed password string begins with the prefix $6$. This identifier indicates that the system is utilizing SHA-512 hashing with key-stretching rounds, ensuring that even if the configuration file is leaked, cleartext passwords cannot be easily reversed via standard rainbow tables.

🔑 2. SSH Interface Hardening (Dropbear Daemon Configuration)
Standard password-based SSH access leaves the router vulnerable to continuous automated credential-stuffing attacks. Shifting to Public-Key Authentication exclusively significantly raises the security posture of the device.

Step 2.1: Generate Asymmetric Key-Pair on Workstation
From the Windows Host terminal (using Git Bash or OpenSSH client), generate a highly secure key-pair:

Bash
ssh-keygen -t rsa -b 4096 -f ~/.ssh/id_rsa_openwrt -C "admin@consultancy.local"
Step 2.2: Import Public Key into OpenWRT
Open the file ~/.ssh/id_rsa_openwrt.pub on the workstation and copy its single-line text content.

Access the OpenWRT LuCI Web Interface, navigate to System ➔ Administration ➔ SSH-Keys.

Paste the string into the text field and click Add Key.

Step 2.3: Restructure the Dropbear Configuration File
To disable password authentication entirely and enforce key-only authentication, modify the Dropbear configuration via CLI or the Web UI:

Bash
# Edit configuration via Unified Configuration Interface (UCI)
uci set dropbear.@dropbear[0].PasswordAuth='0'
uci set dropbear.@dropbear[0].RootPasswordAuth='0'

# Relocate the default SSH management port from 22 to 2222 to mitigate mass automated port scanning
uci set dropbear.@dropbear[0].Port='2222'

# Commit adjustments to system storage and restart the service
uci commit dropbear
/etc/init.d/dropbear restart
Step 2.4: Validate Configuration
Expected Successful Connection: ssh root@5.6.7.8 -p 2222 -i ~/.ssh/id_rsa_openwrt

Expected Rejected Connection (No Port Specified): ssh root@5.6.7.8 ➔ Connection refused

Expected Rejected Connection (Without Private Key): Entry denied automatically without presenting a password prompt.

🚫 3. Attack Surface Reduction (IPv6 Disabling)
Since the local IT consulting firm operates exclusively on an IPv4 structure (5.6.7.0/24), leaving IPv6 daemons running introduces unnecessary protocol overhead and an unmonitored attack surface.

Bash
# Stop the running IPv6 DHCP/Router Advertisement daemon
/etc/init.d/odhcpd stop

# Permanently disable the daemon from initiating during system boot sequences
/etc/init.d/odhcpd disable
Verify via the LuCI System Web GUI under System ➔ Startup that odhcpd is flagged in red as Disabled.

🧱 4. Stateful Firewall Hardening Policies
The OpenWRT firewall (fw4, driven by nftables) must be set up using a "Default Deny" approach. Any traffic that isn't explicitly permitted should be automatically dropped.

The following custom rules must be appended to /etc/config/firewall or configured within the LuCI Network ➔ Firewall ➔ Custom Rules section.

Rule 4.1: Internal Web Service Filtering (HTTP Block)
When auditing network policies or restricting access to local intranet assets:

Plaintext
config rule
    option name 'Block-Internal-HTTP'
    option src 'lan'
    option dest_port '80'
    option proto 'tcp'
    option target 'REJECT'
Verification: Accessing [http://5.6.7.8/student.html](http://5.6.7.8/student.html) from a browser will result in an immediate “Unable to connect / Connection Refused” state.

Rule 4.2: Stealth Mode Execution (Inbound ICMP/Ping Block)
Prevent malicious actors or internal compromised hosts from scanning and mapping active devices across the network segment.

Plaintext
config rule
    option name 'Block-ICMP-Inbound'
    option src 'lan'
    option proto 'icmp'
    option target 'DROP'
Verification: Running ping 5.6.7.8 from the Windows host workstation will result in 100% packet loss (Request timed out), hiding the active interface from discovery tools.

Rule 4.3: Port Restructuring & Management Interface Isolation (Port 81 Restriction)
If an alternate web management instance is hosted on Port 81, it must be locked down to prevent unauthorized manipulation.

Plaintext
config rule
    option name 'Restrict-Port-81'
    option src 'lan'
    option dest_port '81'
    option proto 'tcp'
    option target 'DROP'
📈 5. Network Traffic Auditing & Verification Logs
To confirm that the hardening configurations work as intended, monitor network interactions using packet inspection software (such as Wireshark).

5.1 Cleartext Vulnerability Observation (HTTP Layer)
Run a packet capture on the host interface while interacting with unencrypted HTTP paths.

Notice that the request methods, URIs, and payload responses are completely visible in plain text.

Hardening Takeaway: Move all internal corporate landing directories to HTTPS (TLS v1.2/v1.3) to prevent passive eavesdropping and man-in-the-middle content injections.

5.2 Cryptographic Strength Validation (SSH Layer)
Trace the packet exchange during a connection sequence on Port 2222.

Verify that the protocol exchange explicitly enforces SSH version 2 (SSHv2).

Look closely at the cryptographic handshake parameters to confirm that secure algorithms, such as AES-128-CTR or ChaCha20-Poly1305, are actively protecting session data. This baseline protection keeps critical administrator commands safe from decryption exploits or downgrade attempts.