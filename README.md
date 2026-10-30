# Link Creators Club Web Platform on AWS

A secure, production web server on AWS for The Link Creators Club | London. One Ubuntu EC2 instance runs Nginx inside a custom VPC, served over HTTPS on a custom domain, with network traffic logged to S3.

Live site: [linkcreatorsclub.nuel.site](https://linkcreatorsclub.nuel.site)

---

## 1. What the Project Is

Fast-launched sites often go live with open ports, unmanaged DNS, no TLS, and no view of network traffic. I built this platform to avoid those gaps from day one.

The build covers networking, compute, DNS, the web server, TLS, server hardening, traffic logging, and an external vulnerability scan.

- Completed: July 20, 2026
- Region: us-east-1 (single zone, us-east-1a)
- Status: Live

---

## 2. Architecture Diagram

![Architecture diagram](diagram/architecture.gif)

Traffic flow:

```
User browser
   -> Namecheap DNS (nuel.site)
   -> AWS Internet Gateway
   -> Public subnet 10.0.1.0/24
   -> Security group (ports 22, 80, 443)
   -> EC2 Ubuntu 24.04 + Nginx
   -> /var/www/html
```

VPC Flow Logs capture all traffic in the VPC and ship it to an S3 bucket for audits.

---

## 3. AWS Services and Tools Used

| Area | Tool | Purpose |
| --- | --- | --- |
| Network | Amazon VPC, subnet, Internet Gateway, route table | Isolated network with internet access |
| Compute | Amazon EC2 (Ubuntu 24.04 LTS) | Hosts the web server |
| Firewall | Security groups | Controls inbound traffic |
| Logging | VPC Flow Logs + Amazon S3 | Records network traffic for audits |
| DNS | Namecheap | Points the domain to the server |
| Web server | Nginx | Serves the site |
| TLS | Let's Encrypt + Certbot | Free HTTPS certificates with auto renewal |
| Access | MobaXterm (SSH and SFTP) | Server management and file upload |
| Testing | Pentest-Tools.com, dnschecker.org | Vulnerability scan and DNS checks |

---

## 4. How I Built It

1. Network. I created a custom VPC (10.0.0.0/16) with one public subnet (10.0.1.0/24). I attached an Internet Gateway and routed 0.0.0.0/0 through it.
2. Traffic logging. I turned on VPC Flow Logs for ALL traffic at a 1 minute interval and sent them to an S3 bucket.
3. Firewall. I created a security group allowing only ports 22, 80, and 443.
4. Server. I launched an Ubuntu 24.04 EC2 instance, connected over SSH with MobaXterm, and patched the system before any setup.
5. Web server. I installed Nginx and set it to start on boot.
6. Website. I zipped a customized HTML5 template, uploaded it over SFTP, unzipped it, and replaced the default Nginx page in /var/www/html.
7. Branding. I edited the page title, metadata, and favicon for The Link Creators Club | London.
8. DNS. I added A records in Namecheap for nuel.site, www.nuel.site, and linkcreatorsclub.nuel.site, then confirmed propagation on dnschecker.org.
9. HTTPS. I used Certbot to install Let's Encrypt certificates on all three hostnames and redirect HTTP to HTTPS. I tested auto renewal with `certbot renew --dry-run`.
10. Hardening. I set `server_tokens off` in Nginx so response headers no longer reveal the server version.
11. Testing. I ran an external vulnerability scan on Pentest-Tools.com.

Full commands for every step: [docs/deployment-guide.md](docs/deployment-guide.md)

---

## 5. Problems I Hit and How I Fixed Them

1. HTTPS timed out after the certificate install.
   Cause: the security group had no rule for port 443.
   Fix: added an inbound HTTPS rule.

2. SFTP upload to /var/www/html failed with "Permission denied".
   Cause: the ubuntu user cannot write to the web root.
   Fix: uploaded to the home folder, then copied with `sudo cp`.

3. Certbot asked which domains to secure.
   Fix: kept all three hostnames on one certificate so every address loads over HTTPS.

---

## 6. Result

The site is live over HTTPS on all three hostnames. Certificates renew on their own.

The external scan on Pentest-Tools.com returned a low-risk profile:

| Check | Result |
| --- | --- |
| Critical findings | 0 |
| High or medium findings | 0 |
| Open ports | 22, 80, 443 (expected) |
| Server version in headers | Hidden |

### Limits and Next Steps

This is a single-server build. For a larger production load I would:

- Limit SSH (port 22) to my own IP, or replace SSH with AWS Systems Manager Session Manager. It is open to 0.0.0.0/0 right now.
- Add failover. One instance in one zone has none. I built a highly available version with a load balancer and auto scaling in a separate project.
- Add automated backups with scheduled EBS snapshots or an AMI.
- Add CloudWatch alarms for server health.

---

## Repository Structure

```
.
├── README.md
├── diagram/
│   └── architecture.gif
├── docs/
│   └── deployment-guide.md
└── screenshots/
```

---

## Author

Emmanuel Ibegbu
Cloud Security & DevOps Engineer
