# Ncae-DNS-and-MikroTik

## DNS
### Original Setup
<ins>This command can be used at any time to check your config files</ins>  
```
named-checkconf
```
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
