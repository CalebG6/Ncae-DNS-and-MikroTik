# Ncae-DNS-and-MikroTik

## DNS
### Original Setup
**In Rocky linux add the zones to /etc/named.conf and add the zone files to /var/named**  
On Rocky as well add your ip to ```listen-on port 53 { 127.0.0.1; [add your server ip address]; };``` in named.conf as well as changing listen-on-v6 to { none; };  
First in */etc/named/named.conf.default-zones* add for each domain name and each network IP  */etc/named.conf* on Rocky 8  
```
zones "[domain name]" IN {
  type master;
  file "/path/to/file";
  allow-update { none; };
};

zones "[network of the ip backwards].in-addr.arpa" IN {
  type master;
  file "/path/to/file";
  allow-update { none; };
};
```
For the *"/path/to/file"* create a copy of db.empty/named.empty using sudo and name it and place it where it matches what you put in default-zones.  
For forward lookup do
```
@  IN  SOA  [domain name]  root (
the only thing you need to edit in here is incrementing the Serial by 1)
;
@  IN  NS  [name of the DNS server machine]
[name of DNS server machine]  IN  A [DNS IP] ; this line must be here espcially if webserver is not the DNS server
www  IN A    [ip address of the website]
[add whatever other things you need here following that format]
```
For reverse lookup  
```
@  IN  SOA  [domain name].  root.[domain name]. (
the only thing you need to edit in here is incrementing the Serial by 1)
;
@  IN  NS  [name of the DNS server machine].
[unique indentifier of ip backwards of DNS server]  IN PTR  [DNS hostname.domain name of the website] ; this line must also be here if DNS server != webserver
[unique identifier of ip backwards of webserver]  IN PTR    [beginning of domain/whatever you put in forward that matches to this ip]
[add whatever other things you need here following that format]
```
**DO NOT FORGET THE PERIODS WHERE THEY ARE**  
For Rocky * machines the file will look different, it will look like  
```
$TTL 1D
@       IN      SOA     @ root.localhost.  (
                                0       ; serial
                                1D      ; refresh
                                1H      ; retry
                                1W      ; expire
                                3H )    ; minimum TTL
        NS      @
        A       0.0.0.0
        AAAA    ::0
```
**Change it so that it looks like the above files otherwise you will run into errors**. The serial number follows the same rules.  
Can use this to copy and paste into the file to get correct format (everything after ; is a comment)  
```
$TTL    86400
@       IN      SOA     [domain name here]. root.[domain name here]. (
                              0         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                          86400 )       ; Negative Cache TTL
;
@       IN      NS      [hostname here].
```
Multiple zones for reverse lookup can go in the same file  
In */etc/resolv.conf* add your DNS server above the one that's already there if there is one  
```
nameserver [DNS IP address]
```
Restart the service using  
```
sudo systemctl restart named
```
### DNS Hardening   
In order to protect from DoS attacks and to protect the webservers from unknown IPs we'll add the following to */etc/bind/named.conf.options"*  
```
acl "trusted" {
  // put trusted IPs here
};

options{
// There will be some default stuff here don't touch it
  dnssec-validation auto; //might already be listed

  listen-on-v6 { any; }; //might already be listed

  recursion yes;
  allow-recursion { "trusted"; };
  allow-query { "trusted"; };
  allow-query-cahce { "trusted"; };

  query-source address * port 53; //newer version of bind do not need port 53
  
};
```
In practice this may not help that much because the red team may just be using the scoring routing IP but it's still a good precaution  
Change the named.root.key and named.rfc1912.zones file permission 
```
sudo chmod 040 /etc/named.root.key && sudo chmod 040 /etc/named.rfc1912.zones
```
Create backup of important  
```
sudo tar -zcvf /path/to/backup_name /var/named /etc/named.conf /etc/named.root.key /etc/named.rfc1912.zones /etc/resolv.conf
```
Get files from backup (see debugging for SELinux error)  
```
sudo tar -xzvf /path/to/backup_name -C /
```
### Debugging
<ins>This command can be used at any time to check your config files</ins>  
```
named-checkconf
```
If you're restarting services and the nameserver disappears from resolv.conf check  
```
ls -l /etc/resolv.conf
```
And if it looks like  
```
lrwxrwxrwx 1 root root 39 Jan  1 10:10 /etc/resolv.conf -> ../run/systemd/resolve/stub-resolv.conf
```
Then what you're going to do is:
1. Copy the contents of the file
2. Delete the file using ``` sudo rm /etc/resolv.conf ```
3. Then add your own with nano and paste the contents back in
4. (Optional) you can chattr +i the file  

If SERFAIL permissions may be bad (Rcoky 8)
```
sudo chown -R named:named /var/named
sudo chmod -R 750 /var/named
sudo chown named:named /etc/named.conf
sudo chmod 640 /etc/named.conf
```
If journalctl -xe says that SELinux is having trouble with bind run this command 
```
restorecon -Rv /var/named
```  
Rocky 8 configuration vidoes https://www.youtube.com/watch?v=FeZjRF-aVlc&list=PL291a0KYQZSK6E_1j9xkkieCOi_867pyc  
### DNS Reinstallation (Rocky 8)   
Use this if Red team deletes service/configs
```
sudo dnf remove bind bind-utils -y && sudo dnf autoremove -y && sudo dnf install bind bind-utils -y
```

## MikroTik
### IP Setup
If the router has no addresses run ``` /interfaces print ``` to see what interfaces to use  
Then for both interfaces run  
```
/ip address add address=[ip address] interface=[one of your interfaces]
```
### Port Forwarding
In the search bar of a broswer type in the router's external ip with port 8080 ```[ip address]:8080```
In this homepage scroll down to the interior address and turn on NAT to allow port forwarding then create the port forwarding rules for your services  
Most services use tcp however DNS uses <ins>udp</ins>. Make sure you know what the services uses when formatting.  
### DNS Setup
In the homepage in the external portion you can enter in your DNS server
### Hardening
Some commands to harden the router/network  
Creating a new admin user (remove any other users that were already there)  
```
/user add name=<new_admin_username> password=<new_admin_password> group=full
/user disable admin
/user print
/quit
```
Allow Established & Related Connections  
```
/ip firewall filter add chain=forward connection-state=established,related action=accept comment="Allow Established & Related Connections"
```
Block exterior addresses that are spoofing a private ip
```
/ip firewall filter add chain=input src-address=10.0.0.0/8 in-interface=WAN action=drop comment="Block Bogus Private IPs from WAN"
/ip firewall filter add chain=input src-address=172.16.0.0/12 in-interface=WAN action=drop comment="Block Bogus Private IPs from WAN"
/ip firewall filter add chain=input src-address=192.168.0.0/16 in-interface=WAN action=drop comment="Block Bogus Private IPs from WAN"
```
Port forwaring CLI  
```
/ip firewall filter add chain=forward dst-address=<LAN_IP> protocol=<tcp/udp> dst-port=<INT_PORT> action=accept
```
dst-address should either be the router or the machine but both need to be entered  
Only allow internal network to access web console  
```
/ip firewall filter add chain=input protocol=tcp dst-port=8080 src-address=!192.168.<team#>.0/24 action=drop
```
Allow external ping  
```
/ip firewall filter add chain=input protocol=icmp action=accept
```
More firewall rules to possibly add to drop all direct access to ports that don't need port forwarding  
```
/ip firewall filter
add chain=input protocol=tcp dst-port=80 action=accept comment="Allow HTTP"
add chain=input protocol=tcp dst-port=443 action=accept comment="Allow HTTPS"
add chain=input protocol=tcp dst-port=53 action=accept comment="Allow DNS TCP"
add chain=input protocol=udp dst-port=53 action=accept comment="Allow DNS UDP"
add chain=input in-interface=WAN_INTERFACE action=drop comment="Drop all other external traffic"
```
Stop excessive pings to prevent ping flooding (may need to adjust rate depending on scoring)  
```
/ip firewall filter add chain=input protocol=icmp limit=5,5 action=accept comment="Limit ICMP to prevent abuse"
```
**For all the forward drop commands make sure ports 53,80, and 443 can go out for downloads and scoring**  
```
/ip firewall filter
add chain=input protocol=tcp dst-port=!53,80,8080,5432 action=drop log=yes log-prefix="DROPPED INPUT TCP"
add chain=input protocol=udp dst-port=!53 action=drop log=yes log-prefix="DROPPED INPUT UDP"
/ip firewall filter
add chain=forward protocol=tcp dst-port=!53,80,443,5432 action=drop log=yes log-prefix="DROPPED FORWARD TCP"
add chain=forward protocol=udp dst-port=!53 action=drop log=yes log-prefix="DROPPED FORWARD UDP"
```
