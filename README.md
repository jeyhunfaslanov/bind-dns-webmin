# This guide provides a step-by-step walkthrough for installing, configuring, and securing **BIND DNS Server** with **Webmin** on a Linux server.  
# It is structured for use in production-like lab environments and includes security best practices, zone configuration, and upgrade instructions.

## 1. Prerequisites

### Important Security Note
- Use **.corp, .com, .net, .internal** domains for DNS in business environments.  
  ⚠️ Avoid using **.local**, as it is reserved for **mDNS** and may cause issues.

### Security Recommendations
- **BIND Access Controls**: Use ACLs to limit query access.  
- **Network Infrastructure Security**: Restrict access to DNS ports (`53`, `953`, `10000`) on routers/switches.  
- **Regular Updates**: Keep system updated with latest security patches.  
- **Monitoring**: Enable detailed logging and monitoring.  
- **Physical Security**: Restrict physical access to the server.

---

## 2. System Preparation

### Update the System
```bash
dnf update -y
```

## Configure Static Network Settings
### Check available connections:
```bash
nmcli conn show
```
<img width="586" height="107" alt="f02e09ec-e2ba-4c84-9f2c-981a4f066119_586x107" src="https://github.com/user-attachments/assets/6181ef5f-8418-457b-acf1-7fcf1685ba00" />

Method 1: Using nmcli
```bash
nmcli con mod "ens33" ipv4.addresses 10.0.50.1/24 
nmcli con mod "ens33" ipv4.gateway 10.0.50.254 
nmcli con mod "ens33" ipv4.dns "127.0.0.1 8.8.8.8" 
nmcli con mod "ens33" ipv4.dns-search "mycompany.corp"
nmcli con mod "ens33" ipv4.method manual 
nmcli con up "ens33"
```

Method 2: Editing Config File
```bash
cd /etc/NetworkManager/system-connections/
vim ens33.nmconnection
```
Add:
```bash
[connection]
id=ens33
type=ethernet
interface-name=ens33

[ipv4]
method=manual
addresses=10.0.50.1/24
gateway=10.0.50.254
dns=127.0.0.1;8.8.8.8;
dns-search=mycompany.corp
```
Restart service:
```bash
systemctl restart NetworkManager
ip addr show
ip route show
cat /etc/resolv.conf
```
Disable SELinux
```bash
setenforce 0
sed -i 's/SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config
sed -i 's/SELINUX=permissive/SELINUX=disabled/' /etc/selinux/config
getenforce
```

## 3. Install BIND and Start the Service
```bash
dnf install -y bind bind-utils bind-chroot
systemctl enable --now named
systemctl status named
```

## 4. Edit the Main BIND Configuration File
```bash
vim /etc/named.conf
```
Paste:
```bash
options {
    listen-on port 53 { 127.0.0.1; 10.0.50.1; };
    listen-on-v6 port 53 { ::1; };
    directory       "/var/named";
    dump-file       "/var/named/data/cache_dump.db";
    statistics-file "/var/named/data/named_stats.txt";
    memstatistics-file "/var/named/data/named_mem_stats.txt";
    secroots-file   "/var/named/data/named.secroots";
    recursing-file  "/var/named/data/named.recursing";
    allow-query     { localhost; 10.0.50.0/24; };
    allow-transfer  { localhost; };
    forwarders      { 8.8.8.8; 8.8.4.4; 1.1.1.1; };
    forward         only;
    recursion yes;
    dnssec-enable yes;
    dnssec-validation yes;
    managed-keys-directory "/var/named/dynamic";
    pid-file "/run/named/named.pid";
    session-keyfile "/run/named/session.key";
    include "/etc/crypto-policies/back-ends/bind.config";
};

logging {
    channel default_debug {
        file "data/named.run";
        severity dynamic;
    };
    channel query_log {
        file "/var/log/named/query.log" versions 3 size 100M;
        severity info;
        print-category yes;
        print-severity yes;
        print-time yes;
    };
    category queries { query_log; };
};

zone "." IN {
    type hint;
    file "named.ca";
};

zone "mycompany.corp" IN {
    type master;
    file "/var/named/mycompany.corp.zone";
    allow-update { none; };
};

zone "50.0.10.in-addr.arpa" IN {
    type master;
    file "/var/named/10.0.50.zone";
    allow-update { none; };
};

include "/etc/named.rfc1912.zones";
include "/etc/named.root.key";
```
Create Log Directory
```bash
mkdir -p /var/log/named
chown named:named /var/log/named
```
Create Forward Zone File
```bash
vim /var/named/mycompany.corp.zone
```
```bash
$TTL 86400
@   IN  SOA     dns-01.mycompany.corp. admin.mycompany.corp. (
        2024010101  ; Serial
        3600        ; Refresh
        1800        ; Retry
        1209600     ; Expire
        86400       ; Minimum
)

@   IN  NS      dns-01.mycompany.corp.

dns-01          IN  A   10.0.50.1
www             IN  A   10.0.50.10
gateway         IN  A   10.0.50.254
webserver       IN  CNAME   www.mycompany.corp.
@               IN  MX  10  mail.mycompany.corp.
@               IN  TXT "v=spf1 mx ~all"
```
Create Reverse Zone File
```bash
vim /var/named/10.0.50.zone
```
```bash
$TTL 86400
@   IN  SOA     dns-01.mycompany.corp. admin.mycompany.corp. (
        2024010101
        3600
        1800
        1209600
        86400
)

@   IN  NS      dns-01.mycompany.corp.

1       IN  PTR     dns-01.mycompany.corp.
10      IN  PTR     www.mycompany.corp.
254     IN  PTR     gateway.mycompany.corp.
```
Set Permissions
```bash
chown root:named /var/named/mycompany.corp.zone
chown root:named /var/named/10.0.50.zone
chmod 640 /var/named/mycompany.corp.zone
chmod 640 /var/named/10.0.50.zone
```
Validate Configuration
```bash
named-checkconf
named-checkzone mycompany.corp /var/named/mycompany.corp.zone
named-checkzone 50.0.10.in-addr.arpa /var/named/10.0.50.zone
systemctl restart named
systemctl status named
```

## 5. Webmin Installation
```bash
cat > /etc/yum.repos.d/webmin.repo << EOF
[Webmin]
name=Webmin
baseurl=https://download.webmin.com/download/yum
enabled=1
gpgcheck=1
gpgkey=https://download.webmin.com/jcameron-key.asc
EOF

#Install and check status the service
dnf install -y webmin
systemctl enable --now webmin
systemctl status webmin
```

## 6. DNS Zone and Records Configuration
<img width="1679" height="1100" alt="b1ebbac7-b9b4-470a-8b78-9b5590546f90_1679x1100" src="https://github.com/user-attachments/assets/d9ad2b9d-83e9-469a-8370-f58877384b58" />

Adding New Zones via Webmin
Go to Servers → BIND DNS Server → Create master zone

Fill in:
Domain name: mycompany.corp
Email address: admin@mycompany.corp
Master server: dns-01.mycompany.corp

<img width="1681" height="853" alt="ca3dec77-231b-46fb-ab22-11a30d4ad80b_1681x853" src="https://github.com/user-attachments/assets/2639847a-5db8-48e0-b70c-13dda2abbaac" />

Adding Records via Webmin Dashboard
A Record → Address
CNAME Record → Name Alias
MX Record → Mail Server
TXT Record → Text

<img width="1675" height="851" alt="18ddab34-13a8-45b8-8b00-971407a4a624_1675x851" src="https://github.com/user-attachments/assets/e1ba7766-32d1-407c-b75a-d6986c85f449" />

<img width="1963" height="1068" alt="ebbbb23d-1dc1-4877-9dd0-222173989cfe_1963x1068" src="https://github.com/user-attachments/assets/082c582b-5b9f-4650-ac01-fddb7f1b9e26" />

Adding Records also via CLI
```bash
echo "newhost    IN  A   10.0.50.100" >> /var/named/mycompany.corp.zone
sed -i 's/2024010101/2024010102/' /var/named/mycompany.corp.zone
rndc reload mycompany.corp
```

## 7. Security Configuration
Access Control Lists (ACLs)
```bash
acl "trusted" {
    127.0.0.1;
    10.0.50.0/24;
    10.0.0.0/8;
};

options {
    allow-query { trusted; };
    allow-recursion { trusted; };
    allow-query-cache { trusted; };
};
```
## 8. Webmin Upgrade
```bash
curl -o setup-repos.sh https://raw.githubusercontent.com/webmin/webmin/master/setup-repos.sh
sudo sh setup-repos.sh
sudo dnf upgrade webmin
```

## 9. Change GPG Key Method
Remove Old Key
```bash
rpm -q gpg-pubkey --qf '%{NAME}-%{VERSION}-%{RELEASE}\t%{SUMMARY}\n'
rpm -e gpg-pubkey-id
dnf clean all
dnf clean packages
```
Re-import Key
```bash
rpm --import https://download.webmin.com/jcameron-key.asc
rpm -q gpg-pubkey --qf '%{NAME}-%{VERSION}-%{RELEASE}\n' | grep -i jamie
```
Rebuild Repo Config
```bash
rm -f /etc/yum.repos.d/webmin.repo
cat << 'EOF' | sudo tee /etc/yum.repos.d/webmin.repo
[Webmin]
name=Webmin
baseurl=https://download.webmin.com/download/yum
enabled=1
gpgcheck=1
gpgkey=https://download.webmin.com/jcameron-key.asc
EOF

dnf makecache
dnf upgrade webmin
```
Verify
```bash
rpm -q gpg-pubkey --qf '%{NAME}-%{VERSION}-%{RELEASE}\n' | grep -i jamie
rpm -q webmin
systemctl status webmin
```

🔗 Reference Resources

[https://webmin.com/docs/] Webmin Official Documentation

[https://download.webmin.com/jcameron-key.asc] GPG Key URL

[https://raw.githubusercontent.com/webmin/webmin/master/setup-repos.sh] Repository Setup Script
