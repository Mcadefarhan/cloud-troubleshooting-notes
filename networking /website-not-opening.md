# Website not opening on EC2 (Apache httpd)

## Symptom
Opening `http://<public-ip>` in the browser shows "This site can't be reached" (timeout or connection refused).

## Environment
- AWS EC2 (t3.micro, Amazon Linux)
- Region: ap-southeast-2 (Sydney)
- Default VPC, public subnet
- Web server: Apache (httpd)

## How I reproduced it
Two separate failures, reproduced one at a time:
- **Case A:** Removed the inbound rule for port 80 from the security group
- **Case B:** Stopped Apache (httpd) with `sudo systemctl stop nginx`

## Troubleshooting steps
1. Checked instance state: running, public IP assigned
2. Checked the browser error type (timeout vs refused)
3. **Case A (timeout):**
   - Checked security group: no inbound rule for port 80
   - Verified NACL and route table (Internet Gateway route present)
4. **Case B (refused):**
   - Security group had port 80 open, so network path was fine
   - SSHed in and ran `sudo systemctl status httpd`: inactive (dead)
   - Ran `sudo ss -tlnp | grep :80`: nothing listening
5. Tested on browser: `http://<public-ip>`

## Root cause
- **Case A:** Port 80 not allowed in the security group
- **Case B:** httpd service was stopped, nothing listening on port 80

## Fix
- **Case A:** Added inbound rule: HTTP, port 80, source 0.0.0.0/0
- **Case B:** `sudo systemctl start httpd` and `sudo systemctl enable httpd`

## Verification
```
$ curl -I http://<public-ip>
HTTP/1.1 200 OK
Server: Apache
```
Nginx welcome page loaded in the browser.

## Key learning
| Symptom | Likely cause | Check |
|---|---|---|
| Timeout | Security group / NACL / route table | Network path |
| Connection refused | Service not running | `systemctl status`, `ss -tlnp` |
| 502/503/404 | Web server is reachable but app/config is wrong | Nginx logs, app status |

- Same logic as SSH: timeout = network, refused = service
- A website can fail at 3 layers: network, web server, application. Check them in that order

## Commands used
```
curl -v http://<public-ip>
sudo systemctl status nginx
sudo systemctl start nginx
sudo ss -tlnp | grep :80
sudo tail -f /var/log/nginx/error.log
```
