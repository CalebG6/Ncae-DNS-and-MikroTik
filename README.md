# Ncae-DNS-and-MikroTik

## DNS
### Original Setup
First in */etc/named/named.conf.default-zones* add for each domain name and each network IP  
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
For the *"/path/to/file"* create a copy of db.empty using sudo and name it and place it where it matches what you put in default-zones.  
For forward lookup do
```
@  IN  SOA  [domain name]  root (
the only thing you need to edit in here is incrementing the Serial by 1)
;
@  IN  NS  [name of the DNS server machine]
www  IN A    [ip address of the website]
[add whatever other things you need here following that format]
```
For reverse lookup  
```
@  IN  SOA  [domain name].  root.[domain name]. (
the only thing you need to edit in here is incrementing the Serial by 1)
;
@  IN  NS  [name of the DNS server machine].
[unique identifier of ip backwards]  IN PTR    [beginning of domain/whatever you put in forward that matches to this ip]
[add whatever other things you need here following that format]
```
**DO NOT FORGET THE PERIODS WHERE THEY ARE**  
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
