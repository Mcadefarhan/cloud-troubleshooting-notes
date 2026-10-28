# DNS not resolving on EC2

## Symptom
The instance cannot reach websites by name, but works by IP:

```
[ec2-user@my-ip ~]$ ping -c 3 google.com
ping: google.com: Name or service not known


[ec2-user@my-ip ~]$ ping -c 3 8.8.8.8
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=116 time=0.527 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=116 time=0.559 ms
64 bytes from 8.8.8.8: icmp_seq=3 ttl=116 time=0.551 ms

--- 8.8.8.8 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2101ms
rtt min/avg/max/mdev = 0.527/0.545/0.559/0.013 ms
```

## Environment
- AWS EC2 (t3.micro, Amazon Linux)
- Region: ap-southeast-2 (Sydney)
- VPC with Amazon-provided DNS

## How I reproduced it
Disabled **Enable DNS resolution** in the VPC settings.

## Troubleshooting steps
1. Tested by IP: `ping 8.8.8.8` worked, so routing, security group and Internet Gateway were fine
2. Tested by name: `ping google.com` failed with "Name or service not known"
3. Concluded the problem was name resolution, not connectivity
4. Checked the resolver config: `cat /etc/resolv.conf` (or `resolvectl status`)
5. Tried `dig google.com` and `nslookup google.com`: no answer / timeout
6. Checked the VPC settings: **DNS resolution** was disabled

## Root cause
The VPC had DNS resolution disabled, so the Amazon-provided DNS resolver did not respond to queries from the instance.

## Fix
Enabled **DNS resolution** in VPC settings (Actions → Edit VPC settings).

## Verification
```
[ec2-user@ny-ip ~]$ curl -I https://google.com
HTTP/2 301
location: https://www.google.com/
content-type: text/html; charset=UTF-8
content-security-policy-report-only: object-src 'none';base-uri 'self';script-src 'nonce-hi_jz6mv-a1NnRe65dhUGw' 'strict-dynamic' 'report-sample' 'unsafe-eval' 'unsafe-inline' https: http:;report-uri https://csp.withgoogle.com/csp/gws/other-hp
date: Wed, 28 Oct 2026 06:34:22 GMT
expires: Fri, 27 Nov 2026 06:34:22 GMT
```

## Key learning
- If **IP works but name fails**, it is a DNS problem, not a network problem
- Error wording: "Could not resolve host" / "Name or service not known" = DNS. "Connection timed out" = network path
- Inside a VPC, the Amazon DNS resolver is at the VPC CIDR base + 2 (e.g. `172.31.0.2`) and `169.254.169.253`
- VPC has two settings: **DNS resolution** (resolver works) and **DNS hostnames** (instances get public DNS names). Know the difference
- `NXDOMAIN` means the domain does not exist; a timeout from the resolver means the resolver is unreachable or broken
- Other common causes: wrong DNS server in `/etc/resolv.conf`, custom DHCP option set, wrong Route 53 record, private hosted zone not associated with the VPC

## Commands used
```
ping -c 3 8.8.8.8
ping -c 3 google.com
cat /etc/resolv.conf
resolvectl status
nslookup google.com
dig google.com
```
