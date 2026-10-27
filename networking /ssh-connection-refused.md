# SSH connection refused on EC2

## Symptom
Running the following command returns "Connection refused":

```
$ farha@LAPTOP-46R52580 MINGW64 ~/downloads
$ ssh -i "example-key.pem" ec2-user@my-ip.ap-southeast-2.compute.amazonaws.com
ssh: connect to host ec2-3-26-my-ip.ap-southeast-2.compute.amazonaws.com port 22: Connection refused

```

## Environment
- AWS EC2 (t3.micro, Amazon Linux)
- Region: ap-southeast-2 (Sydney)
- Default VPC, public subnet
- Security group: port 22 open to My IP

## How I reproduced it
Connected to the instance, then stopped the SSH daemon:

`sudo systemctl stop sshd`

## Troubleshooting steps
1. Checked instance state: it was running
2. Checked the security group: port 22 was open to my IP (so not a firewall issue)
3. Noticed the error was "Connection refused", not "Timed out". This means the packet reached the server, but nothing was listening on port 22
4. Opened a session via EC2 Instance Connect (browser) and ran `sudo systemctl status sshd`: service was inactive (dead)
5. Confirmed with `sudo ss -tlnp | grep :22`: no output, so nothing listening on port 22

## Root cause
The `sshd` service was stopped, so no process was listening on port 22.

## Fix
Started the service and enabled it on boot:

```
sudo systemctl start sshd
sudo systemctl enable sshd
```

## Verification
```
$ sudo systemctl status sshd
● [ec2-user@ip-my-ip ~]$ sudo systemctl status sshd
sshd.service - OpenSSH server daemon
     Loaded: loaded (/usr/lib/systemd/system/sshd.service; enabled; preset: enabled)
     Active: active (running) since Tue 2026-10-27 07:08:16 UTC; 1min 55s ago

$  ssh -i "my-example-key.pem" ec2-user@ec2-my-ip.ap-southeast-2.compute.amazonaws.com
** WARNING: connection is not using a post-quantum key exchange algorithm.
** This session may be vulnerable to "store now, decrypt later" attacks.
** The server may need to be upgraded. See https://openssh.com/pq.html
   ,     #_
   ~\_  ####_        Amazon Linux 2023
  ~~  \_#####\
  ~~     \###|
  ~~       \#/ ___   https://aws.amazon.com/linux/amazon-linux-2023
   ~~       V~' '->
    ~~~         /
      ~~._.   _/
         _/ _/
       _/m/'
Last login: Tue Oct 27 07:03:17 2026 from 223.235.99.53
[ec2-user@ip-my-ip ~]$
```

## Key learning
| Error | Meaning | Where to look |
|---|---|---|
| Connection timed out | Packets are dropped before reaching the server | Security group, NACL, route table, public IP |
| Connection refused | Server reached, but nothing listening on the port | Service status, port, OS firewall |

- Timeout = network path problem. Refused = service problem.
- Always check whether the port is open (security group) **and** whether a process is listening (`ss -tlnp`).

## Commands used
```
ssh -v -i my-key.pem ec2-user@<public-ip>
sudo systemctl status sshd
sudo systemctl start sshd
sudo ss -tlnp | grep :22
```
