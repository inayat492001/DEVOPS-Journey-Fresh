FIELD GUIDE / HANDS-ON AWS IMPLEMENTATION

Application Load Balancer + Auto Scaling Group

DevOps assignment • HTTP only • Ubuntu 26.04 LTS • Nginx • t3.micro

<table>
<tbody>
<tr class="odd">
<td><p><strong>ASSIGNMENT FROM YOUR TEAM LEAD</strong></p>
<p>“Deploy a simple Nginx website behind an Application Load Balancer. Start with three Ubuntu EC2 instances, maintain at least three, and allow Auto Scaling to grow to five when CPU usage rises. Show me the healthy targets, website traffic distribution, and evidence of the scaling test.”</p></td>
</tr>
</tbody>
</table>

## Definition of done

  - Three Nginx EC2 targets are healthy in ALB-TG and the website loads
    through ALB-Test on HTTP port 80.

  - Test-ASG launches instances from Test-Launch-Template, using min 3 /
    desired 3 / max 5.

  - A 50% average-CPU target tracking policy is configured; a
    five-minute CPU stress test and scaling activity are documented.

## Fixed resource names and settings

| **Resource**           | **Exact name / value**    | **Resource**       | **Exact name / value** |
| ---------------------- | ------------------------- | ------------------ | ---------------------- |
| **ALB Security Group** | ALB-SG                    | EC2 Security Group | EC2-SG                 |
| **Key pair**           | ALB-Key (.pem)            | Launch Template    | Test-Launch-Template   |
| **Target Group**       | ALB-TG                    | Application LB     | ALB-Test               |
| **Auto Scaling Group** | Test-ASG                  | Instance type      | t3.micro               |
| **Base image**         | Ubuntu Server 26.04 LTS   | Web server         | Nginx, HTTP :80        |
| **ASG group sizes**    | Min 3 / Desired 3 / Max 5 | CPU target         | 50% average            |

# 1 | The architecture

![Architecture: Internet users connect using HTTP port 80 to ALB-Test,
which forwards to ALB-TG. Three Nginx EC2 targets run under Test-ASG.
The Launch Template defines instances and CloudWatch influences
scaling.](./media/image1.png)

Figure 1. Request traffic goes through ALB-Test and ALB-TG; Test-ASG
manages the EC2 fleet. The Launch Template and CloudWatch operate on the
capacity-management side.

<table>
<tbody>
<tr class="odd">
<td><p><strong>THE CRITICAL INTEGRATION</strong></p>
<p>ALB-Test → ALB-TG ← Test-ASG. The ALB listener forwards HTTP requests to ALB-TG. Separately, Test-ASG attaches that same Target Group and automatically registers or deregisters the EC2 instances it manages.</p></td>
</tr>
</tbody>
</table>

# 2 | Shopping mall analogy

Think of a mall with three identical restaurants. The receptionist does
not build restaurants. A manager builds them and keeps the directory
updated.

![Shopping mall analogy: ALB-Test receptionist, ALB-TG restaurant
directory, three EC2 Nginx restaurants, and Test-ASG mall
manager.](./media/image2.png)

Figure 2. ALB-Test is the receptionist, ALB-TG is the restaurant
directory, Nginx servers are restaurants, and Test-ASG is the manager.

| **AWS service**          | **Analogy**          | **Practical responsibility**                             |
| ------------------------ | -------------------- | -------------------------------------------------------- |
| **ALB-Test**             | Receptionist         | Routes incoming HTTP requests to healthy targets.        |
| **ALB-TG**               | Restaurant directory | Registers targets and checks their HTTP health.          |
| **EC2 + Nginx**          | Restaurant and staff | Provides the webpage and responds to requests.           |
| **Test-ASG**             | Mall manager         | Maintains instance count; adds or removes servers.       |
| **Test-Launch-Template** | Building blueprint   | Defines the AMI, size, key, SG and bootstrap commands.   |
| **CloudWatch**           | Workload report      | Supplies utilization measurements for scaling decisions. |
| **Security Groups**      | Entrance guards      | Control allowed HTTP and SSH connections.                |

# 3 | Before you begin

  - Choose one AWS Region and use it for every resource. Confirm that a
    default VPC and at least two default subnets in different
    Availability Zones are available.

  - The default public subnets should assign public IPv4 addresses so
    you can SSH into EC2. Check the subnet setting if public IPs are
    missing.

  - Ensure your account can create EC2, ALB and ASG resources. t3.micro,
    ALB, public IPv4, EBS and detailed monitoring can incur charges.

  - This is intentionally a public-subnet, HTTP-only learning lab. It is
    not a production internet-facing design.

# 4 | Create the resources in AWS Console

## Step 1 — Create ALB-SG

Path: EC2 → Security Groups → Create security group. Name ALB-SG; choose
the default VPC.

| **Direction** | **Type / port**      | **Source / destination**  |
| ------------- | -------------------- | ------------------------- |
| **Inbound**   | HTTP · TCP 80        | Anywhere-IPv4 (0.0.0.0/0) |
| **Outbound**  | Default: All traffic | 0.0.0.0/0                 |

Mall analogy: the main entrance guard allows visitors through HTTP gate
80.

## Step 2 — Create EC2-SG

Path: EC2 → Security Groups → Create security group. Name EC2-SG; choose
the same default VPC.

| **Direction** | **Type / port**      | **Source / destination**        |
| ------------- | -------------------- | ------------------------------- |
| **Inbound**   | SSH · TCP 22         | My IP (IPv4), not Anywhere-IPv4 |
| **Inbound**   | HTTP · TCP 80        | Security Group: ALB-SG          |
| **Outbound**  | Default: All traffic | 0.0.0.0/0                       |

Mall analogy: the restaurant guard admits web visitors only from the
receptionist, but allows you in through the SSH administration door.

## Step 3 — Create ALB-Key

Path: EC2 → Key Pairs → Create key pair. Select RSA, .pem format, and
the name ALB-Key. Save the private file securely. The key is for SSH
administration, not for ALB-to-target HTTP traffic.

## Step 4 — Create Test-Launch-Template

Path: EC2 → Launch Templates → Create launch template.

| **Field**                                             | **Choose**                                                  |
| ----------------------------------------------------- | ----------------------------------------------------------- |
| **Name**                                              | Test-Launch-Template                                        |
| **AMI**                                               | Official Canonical Ubuntu Server 26.04 LTS, x86\_64 / amd64 |
| **Instance type**                                     | t3.micro (2 vCPU / 1 GiB RAM)                               |
| **Key pair**                                          | ALB-Key                                                     |
| **Security Group**                                    | Existing EC2-SG                                             |
| **Storage**                                           | 8 GiB or the AMI-required minimum                           |
| **Subnet**                                            | Do not fix a subnet in the Launch Template                  |
| **Advanced details → Detailed CloudWatch monitoring** | Enable (1-minute CPU metrics; may incur a charge)           |

How to find the exact Ubuntu AMI: browse public AMIs, filter Canonical
owner ID 099720109477 and the Ubuntu 26.04 (resolute) amd64-server
image. AMI IDs vary by Region; do not copy an ID from another Region.

### Step 4A — Paste the User Data script

In Advanced details → User data, paste the following Bash script. This
runs when a new EC2 instance is first launched by ASG.

<table>
<tbody>
<tr class="odd">
<td>#!/bin/bash<br />
set -e<br />
# Update package information<br />
sudo apt update -y<br />
# Install Nginx and stress<br />
sudo apt install nginx stress -y<br />
# Create the demonstration webpage<br />
echo "&lt;h1&gt;Some Content&lt;/h1&gt;" &gt; /var/www/html/index.html<br />
# Identify which EC2 instance answered<br />
echo "&lt;p&gt;Server: $(hostname)&lt;/p&gt;" &gt;&gt; /var/www/html/index.html<br />
# Enable and start Nginx<br />
sudo systemctl enable nginx<br />
sudo systemctl start nginx</td>
</tr>
</tbody>
</table>

<table>
<tbody>
<tr class="odd">
<td><p><strong>WHY THIS SCRIPT IS CORRECT</strong></p>
<p>User Data scripts run as root on Linux EC2, so sudo is redundant but works on Ubuntu. The &gt; redirection writes index.html successfully here because the script itself runs as root. After SSH login as ubuntu, use sudo sh -c for the same protected-file write.</p></td>
</tr>
</tbody>
</table>

### Key commands in the script

| **Command / syntax**                 | **What it does**                              |
| ------------------------------------ | --------------------------------------------- |
| **\#\!/bin/bash**                    | Runs the bootstrap using Bash.                |
| **set -e**                           | Stops on an unhandled failed command.         |
| **sudo apt update -y**               | Refreshes package information.                |
| **sudo apt install nginx stress -y** | Installs the web server and CPU load tool.    |
| **\> / \>\>**                        | Overwrite a file / append to a file.          |
| **$(hostname)**                      | Displays the unique name of the EC2 instance. |
| **systemctl enable / start**         | Enable Nginx at future boots / start it now.  |

Mall analogy: the Launch Template is the standard construction plan.
Every shop built by the manager gets Ubuntu, Nginx, a webpage and the
same access rules.

## Step 5 — Create ALB-TG (Target Group)

Path: EC2 → Target Groups → Create target group.

| **Field**           | **Choose**  |
| ------------------- | ----------- |
| **Target type**     | Instances   |
| **Name**            | ALB-TG      |
| **Protocol / port** | HTTP / 80   |
| **VPC**             | Default VPC |
| **Health check**    | HTTP /      |
| **Success code**    | 200         |

Do not manually register targets. Test-ASG will register instances when
you attach ALB-TG. Mall analogy: this is the receptionist’s directory
and inspection checklist.

## Step 6 — Create ALB-Test and CONNECT it to ALB-TG

Path: EC2 → Load Balancers → Create load balancer → Application Load
Balancer.

| **Field**           | **Choose**                                           |
| ------------------- | ---------------------------------------------------- |
| **Name**            | ALB-Test                                             |
| **Scheme**          | Internet-facing                                      |
| **IP address type** | IPv4                                                 |
| **VPC**             | Default VPC                                          |
| **Subnets**         | At least two default public subnets in different AZs |
| **Security Group**  | ALB-SG                                               |
| **Listener**        | HTTP / 80                                            |
| **Default action**  | Forward to target group ALB-TG                       |

<table>
<tbody>
<tr class="odd">
<td><p><strong>CONNECTION A — EXPLICITLY CONFIGURE IT</strong></p>
<p>ALB-Test → ALB-TG. While creating the ALB, the HTTP :80 listener must have Default action = Forward to ALB-TG. Without this rule, the ALB does not know where to route the requests.</p></td>
</tr>
</tbody>
</table>

At this point ALB-TG may show zero registered targets. That is normal;
the ASG has not been created yet.

## Step 7 — Create Test-ASG and CONNECT it to ALB-TG

Path: EC2 → Auto Scaling Groups → Create Auto Scaling group.

1.  Enter ASG name Test-ASG. Choose existing Launch Template
    Test-Launch-Template and its default version.

2.  Choose the default VPC and two or more default public subnets in
    different Availability Zones. Verify they assign public IPv4
    addresses for SSH.

3.  On the Configure advanced options page, under Load balancing, select
    Attach to an existing load balancer → Choose from your load balancer
    target groups → ALB-TG.

4.  Enable Elastic Load Balancing health checks. Set Health check grace
    period = 180 seconds initially; increase it if first-boot
    installation regularly takes longer.

5.  Set Desired capacity = 3, Minimum capacity = 3, Maximum capacity =
    5. Review and create the ASG.

<table>
<tbody>
<tr class="odd">
<td><p><strong>CONNECTION B — THE SECOND HALF OF THE INTEGRATION</strong></p>
<p>Test-ASG → ALB-TG. ASG uses Test-Launch-Template to launch EC2 instances, then automatically registers them in ALB-TG. It also deregisters them when it removes them. ASG is not itself on the HTTP request path.</p></td>
</tr>
</tbody>
</table>

Mall analogy: the manager opens three restaurants, then adds those
restaurants to the same directory the receptionist uses. No second
directory is needed.

## Step 8 — Confirm the ALB ↔ Target Group ↔ ASG integration

| **Where in AWS Console**                     | **What you must see**                                            |
| -------------------------------------------- | ---------------------------------------------------------------- |
| **ALB-Test → Listeners and rules**           | HTTP :80 forwards to ALB-TG.                                     |
| **Test-ASG → Integrations / Load balancing** | ALB-TG attached to Test-ASG.                                     |
| **Test-ASG → Instance management**           | Three EC2 instances launched.                                    |
| **ALB-TG → Targets**                         | Three registered EC2 instances show Healthy after bootstrapping. |

If you forgot the attachment, go to Test-ASG → Integrations → Load
balancing → Edit and attach ALB-TG. You can enable ELB health checks
under the group’s Details → Health checks → Edit settings.

## Step 9 — Add the automatic CPU scaling policy

Path: EC2 → Auto Scaling Groups → Test-ASG → Automatic scaling → Create
dynamic scaling policy.

| **Setting**         | **Value**                                          |
| ------------------- | -------------------------------------------------- |
| **Policy type**     | Target tracking                                    |
| **Metric**          | Average CPU utilization (ASGAverageCPUUtilization) |
| **Target value**    | 50%                                                |
| **Instance warmup** | 180 seconds (initial lab choice)                   |

Mall analogy: the manager receives utilization reports and aims to keep
the average restaurant workload near 50%. Scale-out is policy-driven,
not instantaneous.

# 5 | Validate the deployment

## Step 10 — Check the webpage through the ALB

Path: EC2 → Load Balancers → ALB-Test → Details. Copy the DNS name and
open the HTTP URL (not HTTPS).

|                          |
| ------------------------ |
| http://YOUR-ALB-DNS-NAME |

Expected HTML (hostname varies by instance):

<table>
<tbody>
<tr class="odd">
<td>&lt;h1&gt;Some Content&lt;/h1&gt;<br />
&lt;p&gt;Server: ip-172-31-20-105&lt;/p&gt;</td>
</tr>
</tbody>
</table>

From a Bash-capable local terminal, send several separate HTTP requests
to make changes in the responding hostname easier to observe:

<table>
<tbody>
<tr class="odd">
<td>for i in {1..15}; do<br />
curl -s http://YOUR-ALB-DNS-NAME<br />
echo<br />
done</td>
</tr>
</tbody>
</table>

Seeing different hostnames proves more than one backend served requests.
If the hostname does not change immediately, that alone does not prove
the ALB is broken; also inspect all three healthy targets.

## Step 11 — Connect to EC2, verify Nginx and run stress

Open a terminal on your computer. Connect to each of the three original
instances using its own public IPv4 address:

|                                         |
| --------------------------------------- |
| ssh -i ALB-Key.pem ubuntu@EC2-PUBLIC-IP |

The stress command is entered in your computer’s terminal after SSH
connects, but it runs on the remote EC2 CPU, not on your laptop.

<table>
<tbody>
<tr class="odd">
<td>sudo systemctl status nginx<br />
curl http://localhost<br />
stress --version</td>
</tr>
</tbody>
</table>

If Nginx is active, the local curl command should show your HTML. If
stress is missing, run sudo apt install stress -y.

<table>
<tbody>
<tr class="odd">
<td><p><strong>RUN ON EC2-1, EC2-2 AND EC2-3</strong></p>
<p>stress --cpu 2 --timeout 300s — run this separately in three SSH sessions. It starts two CPU workers for five minutes per instance. In a second SSH session, run top to watch utilization; press q to exit.</p></td>
</tr>
</tbody>
</table>

|                               |
| ----------------------------- |
| stress --cpu 2 --timeout 300s |

Why all three? The ASG evaluates the group’s average CPU utilization. If
only one of three servers is at 100% and the others are idle, the group
average is roughly 33%, below the 50% target.

### Optional: use systemctl to observe a temporary stress service

Nginx is a persistent systemd service, while stress normally runs as a
command. If you want to manage stress using systemctl, use this
alternative on an instance instead of starting a second stress process:

<table>
<tbody>
<tr class="odd">
<td>sudo systemd-run --unit=cpu-stress /usr/bin/stress --cpu 2 --timeout 300s<br />
sudo systemctl status cpu-stress<br />
# Optional early stop:<br />
sudo systemctl stop cpu-stress</td>
</tr>
</tbody>
</table>

## Step 12 — Watch Test-ASG Activity and target health

Check EC2 → Auto Scaling Groups → Test-ASG → Activity and Instance
management; then open EC2 → Target Groups → ALB-TG → Targets.

![Scaling test: three existing EC2 instances run before and during CPU
stress; Test-ASG may add EC2-4 and EC2-5.](./media/image3.png)

Figure 3. Stressing the existing instances can lead to scale-out, but
five minutes does not guarantee a scaling event.

Look for CPU metrics rising, a scaling activity such as launching an
instance, the new instance registering with ALB-TG, and a Healthy target
status. Capacity may increase to four or five but will not exceed the
configured maximum.

<table>
<tbody>
<tr class="odd">
<td><p><strong>DO NOT CONFUSE TWO TESTS</strong></p>
<p>Local stress raises CPU inside EC2; it does NOT generate traffic through the ALB. curl against the ALB DNS demonstrates request distribution. CloudWatch + Test-ASG Activity demonstrates Auto Scaling decisions.</p></td>
</tr>
</tbody>
</table>

With t3.micro burstable CPU credits and metric/warmup delays, a
300-second run might end before scale-out occurs. Inspect CPU graphs and
scaling activity rather than assuming failure. For a deterministic
integration check at low CPU, temporarily change ASG desired capacity
from 3 to 4, verify the fourth target becomes Healthy, and restore it to
3 afterward.

# 6 | Troubleshooting playbook

| **Symptom**                       | **First checks / command**                                                   |
| --------------------------------- | ---------------------------------------------------------------------------- |
| **ALB DNS does not load**         | ALB-Test listener HTTP :80; ALB-SG; correct default public subnets.          |
| **Targets stay Unhealthy**        | Nginx service, EC2-SG HTTP from ALB-SG, target port 80 and health path /.    |
| **Nginx not responding**          | sudo systemctl status nginx; curl http://localhost                           |
| **Bootstrapping failed**          | sudo tail -n 50 /var/log/cloud-init-output.log                               |
| **Cannot SSH**                    | Correct public IP, ALB-Key.pem, ubuntu username, SSH from your current IP.   |
| **No new instances after stress** | CloudWatch CPU metrics, target tracking policy, warmup, max 5, ASG Activity. |
| **HTML not changing**             | Browser/cache effects; use curl, compare hostnames, verify TG targets.       |

Manual editing of a single EC2 instance does not update the Launch
Template. Newly launched replacement instances still use the saved User
Data script. Update the template and ASG template version if you want
future instances to use different bootstrap instructions.

# 7 | What to show the team lead

| **Evidence**           | **Expected observation**                                                          |
| ---------------------- | --------------------------------------------------------------------------------- |
| **Security Groups**    | ALB-SG HTTP 80 from IPv4; EC2-SG SSH from My IP and HTTP from ALB-SG.             |
| **Launch Template**    | Official Ubuntu 26.04 + t3.micro + ALB-Key + Nginx User Data.                     |
| **ALB integration**    | ALB-Test HTTP listener forwards to ALB-TG.                                        |
| **ASG integration**    | Test-ASG attaches ALB-TG and manages three EC2 instances.                         |
| **Health and routing** | Three Healthy targets; Nginx content served through ALB DNS.                      |
| **Scaling evidence**   | Five-minute stress attempted on three EC2s; CloudWatch and ASG Activity recorded. |

# 8 | Interview-ready explanation

**INTERVIEW QUESTION | Can you explain your AWS ALB and Auto Scaling
project, including exactly how ALB-Test, ALB-TG and Test-ASG connect?**

<table>
<tbody>
<tr class="odd">
<td><p><strong>EXAMPLE ANSWER AFTER COMPLETING THE LAB</strong></p>
<p>“I deployed a simple Nginx website using an AWS Application Load Balancer and Auto Scaling Group. ALB-Test accepts HTTP traffic on port 80 and forwards it to ALB-TG. Test-ASG attaches the same target group, launches three Ubuntu 26.04 t3.micro instances from Test-Launch-Template, and maintains 3–5 instances. I enabled detailed CPU monitoring and a 50% target tracking policy. I verified target health, checked HTTP responses through the ALB DNS, and ran a five-minute CPU stress test while inspecting CloudWatch and scaling activity.”</p></td>
</tr>
</tbody>
</table>

### Quick interview follow-up questions

**Q1. Why use a Target Group between ALB and ASG?**

ALB-Test forwards HTTP requests to ALB-TG. Test-ASG attaches that same
target group and automatically registers or deregisters its EC2
instances. The two services integrate through ALB-TG.

**Q2. What is the difference between ALB and ASG?**

ALB selects a healthy target for each HTTP request; ASG maintains and
changes the number of EC2 instances according to capacity settings,
health checks, and scaling policies.

**Q3. Why is a Launch Template required?**

It provides the reproducible EC2 configuration: Ubuntu 26.04 AMI,
t3.micro, ALB-Key, EC2-SG, detailed monitoring, and the Nginx / stress
User Data script.

**Q4. Why might five minutes of stress not launch another EC2
instance?**

CloudWatch aggregation, scaling-policy evaluations, warmup timing, and
T3 CPU credits can delay or limit observed scale-out. Check Test-ASG
Activity and CPU metrics; do not assume the configuration failed.

Only say that scale-out actually occurred if you observed a new instance
launch and register successfully.

# 9 | Cost awareness and cleanup

This lab may incur charges for up to five EC2 instances, the ALB, EBS
storage, public IPv4 addresses, detailed monitoring and T3 Unlimited
surplus CPU credits. Do not leave it running after practice.

1\. Delete Test-ASG and wait for ASG-managed instances to terminate.

2\. Delete ALB-Test and ALB-TG.

3\. Delete Test-Launch-Template, then the lab Security Groups ALB-SG and
EC2-SG (when no longer attached).

4\. Verify there are no leftover billable lab resources. Keep the
default VPC and its subnets.

## 15-point project completion checklist

Use this complete 15-point checklist to verify each AWS resource, record
evidence for your team lead, and confirm cleanup when the lab ends.

| **Check**                           | **Proof to capture**                                                                              |
| ----------------------------------- | ------------------------------------------------------------------------------------------------- |
| \[ \] 1. ALB Security Group         | ALB-SG inbound HTTP TCP 80 from Anywhere-IPv4 (0.0.0.0/0).                                        |
| \[ \] 2. EC2 Security Group         | EC2-SG SSH TCP 22 from My IP; HTTP TCP 80 from ALB-SG.                                            |
| \[ \] 3. SSH key pair               | ALB-Key (RSA, .pem) created and stored safely.                                                    |
| \[ \] 4. Launch Template            | Test-Launch-Template uses official Ubuntu 26.04 LTS and t3.micro.                                 |
| \[ \] 5. Automated software setup   | User Data installed Nginx and stress, created HTML, and enabled/started Nginx.                    |
| \[ \] 6. Target Group               | ALB-TG uses HTTP port 80; health check path / and success code 200.                               |
| \[ \] 7. Load Balancer integration  | ALB-Test listener HTTP :80 forwards to ALB-TG.                                                    |
| \[ \] 8. Auto Scaling configuration | Test-ASG uses Test-Launch-Template; min 3, desired 3, max 5.                                      |
| \[ \] 9. Three healthy targets      | Three ASG-managed instances are registered as Healthy in ALB-TG.                                  |
| \[ \] 10. ALB website test          | Opening http://ALB-DNS-NAME displays the Nginx demonstration page.                                |
| \[ \] 11. Request distribution      | Multiple ALB requests show different server hostnames (when observed).                            |
| \[ \] 12. CPU target tracking       | 50% average CPU policy configured; detailed monitoring enabled.                                   |
| \[ \] 13. Stress test               | stress --cpu 2 --timeout 300s run on each of the original three EC2 instances.                    |
| \[ \] 14. Scaling observations      | CloudWatch CPU graphs, Test-ASG Activity, capacity and target health inspected; results recorded. |
| \[ \] 15. Resource cleanup          | Test-ASG, ALB-Test, ALB-TG and lab resources cleaned up; default VPC retained.                    |

# 10 | Official reference documentation

Use these primary sources if a console label changes or you want to
check an AWS behavior:

**AWS — Attach a load balancer to an ASG**
https://docs.aws.amazon.com/autoscaling/ec2/userguide/attach-load-balancer-asg.html

**AWS — ALB creation and listener setup**
https://docs.aws.amazon.com/elasticloadbalancing/latest/application/create-application-load-balancer.html

**AWS — Target tracking scaling policy**
https://docs.aws.amazon.com/autoscaling/ec2/userguide/as-scaling-target-tracking.html

**AWS — Detailed monitoring for Auto Scaling instances**
https://docs.aws.amazon.com/autoscaling/ec2/userguide/enable-as-instance-metrics.html

**Canonical — Find official Ubuntu AMIs on AWS**
https://ubuntu.com/aws/docs/aws-how-to/instances/find-ubuntu-images/
