# 🌀 Highly Available WordPress on AWS

This project deploys a **highly available and scalable WordPress website** on AWS using:

* 🚀 EC2 (Amazon Linux 2)
* 🧠 RDS (MySQL)
* 📆 S3 (media storage with WP Offload Media)
* ⚖️ ALB + Auto Scaling Group
* 📊 CloudWatch (Auto Scaling policies)
* 🔐 IAM, Security Groups, etc.

---

## 📁 Architecture

```
Internet
   ↓
Route 53 (Optional)
   ↓
Application Load Balancer (ALB)
   ↓
Auto Scaling Group (ASG)
   ↓
EC2 instances (Amazon Linux 2 + WordPress)
 ↙️              ↘
RDS (MySQL)     S3 (Offloaded media)
```

---

## ⚙️ Services Used

| Service    | Purpose                             |
| ---------- | ----------------------------------- |
| EC2        | WordPress server                    |
| ALB        | Distributes traffic to EC2s         |
| ASG        | Launches EC2s across AZs            |
| RDS        | WordPress DB (multi-AZ optional)    |
| S3         | Stores WordPress media              |
| IAM        | EC2 role for S3 access              |
| CloudWatch | Metrics + alarms + scaling triggers |

---

## 🛠️ Launch Template - User Data

```bash
#!/bin/bash
# Update system
dnf update -y

# Install Apache, PHP, and MySQL client
dnf install -y httpd php php-mysqli php-fpm php-opcache php-gd php-curl php-mbstring php-xml php-json mariadb105-server

# Start and enable Apache
systemctl start httpd
systemctl enable httpd

# Fix Apache DirectoryIndex to prioritize index.php over index.html
sed -i 's/DirectoryIndex index.html/DirectoryIndex index.php index.html/' /etc/httpd/conf/httpd.conf


# Download and configure WordPress
cd /var/www/html
wget https://wordpress.org/latest.zip
dnf install -y unzip
unzip latest.zip
cp -r wordpress/* .
rm -rf wordpress latest.zip

# Set proper permissions
chown -R apache:apache /var/www/html
chmod -R 755 /var/www/html

# Create wp-config.php
cat > /var/www/html/wp-config.php <<EOF
<?php
define('DB_NAME', 'wordpress_db');
define('DB_USER', 'wp_user');
define('DB_PASSWORD', 'StrongPass123!');
define('DB_HOST', '<RDS-endpoint> or localhost');
define('DB_CHARSET', 'utf8');
define('DB_COLLATE', '');
define('AUTH_KEY',         'replace_this');
define('SECURE_AUTH_KEY', 'replace_this');
define('LOGGED_IN_KEY',   'replace_this');
define('NONCE_KEY',       'replace_this');
define('AUTH_SALT',       'replace_this');
define('SECURE_AUTH_SALT','replace_this');
define('LOGGED_IN_SALT',  'replace_this');
define('NONCE_SALT',      'replace_this');
\$table_prefix = 'wp_';
define('WP_DEBUG', false);
if ( !defined('ABSPATH') ) define('ABSPATH', dirname(__FILE__) . '/');
require_once(ABSPATH . 'wp-settings.php');
EOF


sed -i '/WP_DEBUG/a define("WP_HOME","http://<ALB-endpoint>");\ndefine("WP_SITEURL","http://<ALB-endpoint>");' /var/www/html/wp-config.php

# Restart Apache to apply
systemctl restart httpd


```

---

## 📆 RDS Configuration

* Engine: MySQL
* Multi-AZ: Enabled (optional)
* Public access: No
* Security Group:

  * Inbound: MySQL (3306) from EC2 SG
* DB Name: `wordpress_db`
* Username: `wp_user`
* Password: `StrongPass123!`

Update `wp-config.php`:

```php
define( 'DB_NAME', 'wordpress_db' );
define( 'DB_USER', 'wp_user' );
define( 'DB_PASSWORD', 'StrongPass123!' );
define( 'DB_HOST', 'your-rds-endpoint.rds.amazonaws.com' );
```

---

## 🖼️ S3 Media Offload (Optional but Recommended)

### IAM Role for EC2

```json
{
	"Version": "2012-10-17",
	"Statement": [
		{
			"Sid": "AllowS3ReadOnlyAccessToMedia",
			"Effect": "Allow",
			"Principal": "*",
      "Action": [
        "s3:PutObject",
        "s3:GetObject",
        "s3:DeleteObject",
        "s3:ListBucket"
      ],
			"Resource": "arn:aws:s3:::your-bucket-name/wp-content/uploads/*"
		}
	]
}
```

### WordPress Plugin

1. Install **WP Offload Media Lite** and configure it.
2. Connect with your bucket to offload images
3. Confirm uploaded media appears in S3
4. Disable : Block all public access. If you are not using AWS CloudFront as CDN.

---

## 📊 CloudWatch Alarms & Auto Scaling

### Enable Group Metrics

* Go to EC2 → Auto Scaling Group → Monitoring
* Enable **Group Metrics Collection** (1-minute granularity)

### Create Alarm

1. Go to CloudWatch → Alarms → Create Alarm
2. Choose:

   * **Metric** → Auto Scaling → Group Metrics → `GroupAverageCPUUtilization`
3. Alarm if CPU > 70% for 5 min

### Attach Scaling Policy

* Go to Auto Scaling Group → Automatic Scaling
* Create:

  * **Scale Out Policy**: Add 1 instance if CPU > 70%
  * **Scale In Policy**: Remove 1 instance if CPU < 30%

---

## 🔐 Security Groups

### ALB SG:

* Inbound: 80, 443, 22 from `0.0.0.0/0`
* Outbound: All traffic to `0.0.0.0/0`

### EC2 Launch Template SG:

* Inbound:

  * 22 from anywhere
  * 80/443 from ALB SG
* Outbound: All traffic

### RDS SG:

* Inbound: 3306 from EC2 SG
* Outbound: All traffic

---

## ✅ Deployment Flow Summary

1. Create RDS (MySQL) → Note endpoint
2. Create S3 bucket → Keep public access open for now
3. Create IAM Role for EC2 → Attach S3 policy
4. Create Launch Template with User Data
5. Create ALB + Target Group
6. Create ASG with Launch Template → Register with ALB
7. Configure WordPress (`wp-config.php`) to use RDS
8. Install WP Offload Media Plugin
9. Enable CloudWatch + Auto Scaling policies

---

## 💬 To-Do / Optional Add-ons

* [ ] Add HTTPS with ACM and attach to ALB
* [ ] Use Route 53 domain (Alias to ALB DNS)
* [ ] Install CloudWatch Agent for memory/disk monitoring
* [ ] Replace EC2 with ECS Fargate + EFS
* [ ] Automate all this via Terraform or CloudFormation

---

## 🧐 Author

**Akshat Verma**
DevOps Engineer | Cloud Enthusiast | AWS Certified

---
