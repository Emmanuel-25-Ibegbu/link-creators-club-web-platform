Deployment Guide: Link Creators Club Web Platform

This guide walks through the full build in order. Each step says what I did and why.

Resource IDs, IP addresses, and key names are masked. Replace the placeholders with your own values.

Prerequisites
An AWS account with permission to create VPC and EC2 resources
A domain name (I used nuel.site from Namecheap)
An SSH client with SFTP (I used MobaXterm on Windows)
Website files ready on your computer
1. Build the Network

All resources go in us-east-1, zone us-east-1a.

1.1 Create the VPC
Name: nuel.com-vpc.001
IPv4 CIDR: 10.0.0.0/16

Why: the VPC gives the project its own private network on AWS.

1.2 Create the public subnet
Name: nuel-public-subnet-1a
Zone: us-east-1a
CIDR: 10.0.1.0/24

Why: the web server sits here so it can reach and be reached from the internet.

1.3 Create and attach the Internet Gateway

Create an Internet Gateway and attach it to nuel.com-vpc.001.

Why: without it, nothing in the VPC can talk to the internet.

1.4 Configure the route table
Add route: destination 0.0.0.0/0, target the Internet Gateway
Associate the route table with nuel-public-subnet-1a

Why: this sends internet-bound traffic from the subnet out through the gateway.

1.5 Turn on VPC Flow Logs
Filter: ALL
Aggregation interval: 1 minute
Destination: an S3 bucket
Log file format: Hive-compatible S3 prefixes

Why: flow logs record accepted and rejected traffic for audits and incident review.

2. Create the Security Group
Name: nuel.com-vpc.001-securitygroup
VPC: nuel.com-vpc.001

Inbound rules:

Port	Protocol	Source	Purpose
22	SSH	0.0.0.0/0	Remote server management
80	HTTP	0.0.0.0/0	First web connections, redirected to HTTPS
443	HTTPS	0.0.0.0/0	Encrypted site traffic

Outbound: default, all traffic allowed.

Recommended: set the port 22 source to "My IP" instead of 0.0.0.0/0. This cuts SSH exposure to one address.

3. Launch the EC2 Instance
AMI: Ubuntu Server 24.04 LTS
Subnet: nuel-public-subnet-1a
Auto-assign public IP: Enabled
Security group: nuel.com-vpc.001-securitygroup
Key pair: new RSA key pair, .pem format

Download the key file and store it safely. AWS will not show it again.

4. Connect and Update the Server

Connect from MobaXterm with SSH:

Host: the instance public IP
Username: ubuntu
Private key: your .pem file

Update the system:

bash
sudo apt update -y
sudo apt upgrade -y
sudo apt clean && sudo apt autoremove -y
sudo reboot

Reconnect after the reboot and confirm the new kernel:

bash
uname -r

Why: patching first closes known security holes before the server goes public.

5. Install Nginx
bash
sudo apt install nginx -y
sudo systemctl start nginx
sudo systemctl enable nginx

Open http://<public-ip> in a browser. You should see the Nginx welcome page.

Why: enable makes Nginx start on its own after every reboot.

6. Deploy the Website
6.1 Upload the files

Zip the site files on your computer as webcontent.zip. Then on the server:

bash
mkdir ~/webcontent

Drag webcontent.zip into ~/webcontent with the MobaXterm SFTP panel.

6.2 Unzip
bash
cd ~/webcontent
sudo apt install unzip -y
unzip webcontent.zip
6.3 Replace the default page
bash
sudo rm -rf /var/www/html/*
sudo cp -rf ~/webcontent/* /var/www/html/

Reload the public IP in your browser. Your site should load.

7. Brand the Site

Edit the page title and metadata:

bash
sudo nano /var/www/html/index.html

Updated head block:

html
<title>The Link Creators Club | London</title>
<link rel="icon" type="image/png" href="favicon.png">
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width, initial-scale=1, user-scalable=no" />
Add the favicon

Dragging files straight into /var/www/html/ fails with "Permission denied". The ubuntu user cannot write there. Workaround:

Drag favicon.png into /home/ubuntu/.
Copy it with root rights:
bash
sudo cp ~/favicon.png /var/www/html/
sudo systemctl restart nginx
8. Configure DNS

In Namecheap, open Advanced DNS for nuel.site and add:

Type	Host	Value	TTL
A Record	@	EC2 public IP	Automatic
A Record	www	EC2 public IP	Automatic
A Record	linkcreatorsclub	EC2 public IP	Automatic

Check propagation on dnschecker.org before moving on. Certbot fails if DNS does not resolve yet.

Note: the public IP changes if you stop and start the instance. Attach an Elastic IP to keep it fixed.

9. Enable HTTPS
bash
sudo apt install certbot python3-certbot-nginx -y
sudo certbot --nginx -d nuel.site -d www.nuel.site -d linkcreatorsclub.nuel.site

Keep all three hostnames on the certificate and choose to redirect HTTP to HTTPS.

If HTTPS times out, check the security group for an inbound rule on port 443. That was the cause in my build.

Test automatic renewal:

bash
sudo certbot renew --dry-run

Expected result: all simulated renewals succeed.

10. Hide the Nginx Version
bash
sudo nano /etc/nginx/nginx.conf

Inside the http block, set:

nginx
server_tokens off;

Test and apply:

bash
sudo nginx -t
sudo systemctl restart nginx

Why: hiding the version stops attackers from matching your server to known exploits.

11. Verify
Open https://linkcreatorsclub.nuel.site. The padlock should show.
Open http://nuel.site. It should redirect to HTTPS.
Check headers:
bash
curl -I https://linkcreatorsclub.nuel.site

The Server line should read nginx with no version number.

Run an external scan. My Pentest-Tools.com scan returned 0 critical, 0 high, and 0 medium findings, with ports 22, 80, and 443 open as expected.
Confirm flow log files appear in the S3 bucket.
Troubleshooting
Problem	Cause	Fix
HTTPS times out	No inbound rule for port 443	Add HTTPS rule to the security group
SFTP "Permission denied" in /var/www/html	ubuntu user lacks write rights	Upload to home folder, then sudo cp
Certbot fails domain validation	DNS not propagated yet	Wait, then check on dnschecker.org
Site down after instance restart	Public IP changed	Update A records or use an Elastic IP
Cleanup

To avoid charges after testing:

Terminate the EC2 instance.
Release any Elastic IP.
Delete the flow log and empty the S3 bucket.
Delete the security group, subnet, route table, Internet Gateway, and VPC.
Remove the DNS records in Namecheap.