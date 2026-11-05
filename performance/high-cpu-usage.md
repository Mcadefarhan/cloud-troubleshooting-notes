# High CPU usage on EC2

## Symptom
The instance is slow to respond and CloudWatch shows CPU utilization near 100%.

```
$ uptime
 16:58:07 up 4 min,  1 user,  load average: 0.08, 0.02, 0.01



```

<img width="898" height="360" alt="Screenshot 2026-11-05 222849" src="https://github.com/user-attachments/assets/4c094290-1bad-45a5-9c71-52b95c12b515" />


*top output with one process consuming all CPU*

## Environment
- AWS EC2 (t3.micro, Amazon Linux 2023)
- Region: ap-southeast-2 (Sydney)
- Default CloudWatch basic monitoring (5-minute interval)

## How I reproduced it
Started CPU-heavy background processes:

```
yes > /dev/null &
```

## Troubleshooting steps
1. Checked the CloudWatch **CPUUtilization** graph: CPU was close to 100%
2. SSHed in and ran `uptime`: load average was high (about equal to the number of vCPUs)
3. Ran `top`: the `yes` processes were at the top using ~100% CPU
4. Confirmed with `ps aux --sort=-%cpu | head -5`: gave the PIDs and owner of the processes
5. Concluded it was a runaway process, not a network or disk problem

## Root cause
Background `yes` processes were consuming all available CPU.

## Fix
Stopped the processes:

```
 pkill yes
[1]+  Terminated              yes > /dev/null

```

## Verification
```
$ top
%Cpu(s):  0.3 us,  0.2 sy, ... 99.4 id
```
<img width="897" height="460" alt="image" src="https://github.com/user-attachments/assets/07857f01-6423-4951-9583-4b6949a53725" />



*CPU utilization returning to normal after killing the processes*

## Key learning
- Start from the symptom layer: CloudWatch shows **that** CPU is high, `top` shows **which process** and why
- Load average vs vCPU count: load around or above the vCPU count means the CPU is saturated
- Basic monitoring has a 5-minute delay, so graphs lag behind what you see in `top`
- CloudWatch does not show memory or disk usage by default (needs the CloudWatch agent)
- t3 instances are burstable: sustained high CPU can exhaust CPU credits (check `CPUCreditBalance`) and cause throttling or extra charges
- Long-term fixes: find the cause (app bug, traffic spike), resize the instance, or use Auto Scaling

## Commands used
```
uptime
top
ps aux --sort=-%cpu | head -5
pkill yes
```
