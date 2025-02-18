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
  allow--update { none; };
};
```
For the *"/path/to/file"* create a copy of db.empty using sudo and name it and place it where it matches what you put in default-zones.  
For forward lookup do
```
$TTL  86400
@  IN  SOA  [domain name]  root (
the only thing you need to edit in here is incrementing the **Serial** by 1)
;
@  IN  NS  [name of the DNS server machine]
www  IN A    [ip address of the website]
[add whatever other things you need here following that format]
```
For reverse lookup  
```
$TTL  86400
@  IN  SOA  [domain name].  root.[domain name]. (
the only thing you need to edit in here is incrementing the **Serial** by 1)
;
@  IN  NS  [name of the DNS server machine].
[unique identifier of ip backwards]  IN PTR    [beginning of domain/whatever you put in forward that matches to this ip]
[add whatever other things you need here following that format]
```
