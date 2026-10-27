
# SSH connection timeout on EC2

## Symptom
Running  ssh -i "my-example-key.pem" ec2-user@ec2-3-example-159-231.ap-southeast-2.compute.amazonaws.com returns
"ssh: connect to host ec2-3-26-159-231.ap-southeast-2.compute.amazonaws.com port 22: Connection timed out".

## Environment
- AWS EC2 (t3.micro, Amazon Linux)
- Default VPC, public subnet

## How I reproduced it
Deleted the inbound rule for port 22 from the security group.

## Troubleshooting steps
1. Checked instance state: it was running
2. Checked public IP: it was assigned
3. Checked the security group: the port 22 rule was missing
4. (Also verified other components: NACL, route table)

## Root cause
Port 22 was not open in the security group.

## Fix
Added an inbound rule: SSH, port 22, source = My IP.

## Key learning
- A timeout usually points to a firewall or security group issue
- "Connection refused" usually means the service is not running on the server

## Commands used
`ssh -v -i key.pem ec2-user@<ip>`
