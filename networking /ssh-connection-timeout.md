# SSH connection timeout on EC2

## Symptom
Running the following command returns "Connection timed out":

```
$ ssh -i "my-key.pem" ec2-user@ec2-<public-ip>.ap-southeast-2.compute.amazonaws.com
ssh: connect to host ec2-<public-ip>.ap-southeast-2.compute.amazonaws.com port 22: Connection timed out
```

## Environment
- AWS EC2 (t3.micro, Amazon Linux)
- Region: ap-southeast-2 (Sydney)
- Default VPC, public subnet

## How I reproduced it
Deleted the inbound rule for port 22 from the security group.

## Troubleshooting steps
1. Checked instance state: it was running
2. Checked public IP: it was assigned
3. Checked the security group: the port 22 rule was missing
4. Verified the NACL allows inbound/outbound traffic on port 22 and the route table has a route to the Internet Gateway

## Root cause
Port 22 was not open in the security group.

## Fix
Added an inbound rule: SSH, port 22, source = My IP.

## Verification
Retried the SSH command and the connection succeeded:

```
$ ssh -i "my-key.pem" ec2-user@ec2-<public-ip>.ap-southeast-2.compute.amazonaws.com
[ec2-user@ip-<private-ip> ~]$
```

## Key learning
- A timeout usually points to a firewall or security group issue
- "Connection refused" usually means the service is not running on the server

## Commands used
`ssh -v -i my-key.pem ec2-user@<public-ip>`
