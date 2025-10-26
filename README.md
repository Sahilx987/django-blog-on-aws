**# 🏗️ Django Blog Deployment on AWS (Production Setup)

## 🔍 Project Overview
This project demonstrates **end-to-end deployment of a Django web application on AWS**, using production-ready components — **EC2**, **RDS**, **Nginx**, **Gunicorn**, **Route53**, and **Let’s Encrypt SSL**.  
The setup follows industry best practices for scalability, performance, and security.

---

## ⚙️ Architecture Overview
**AWS Infrastructure**
```
VPC: blog-vpc (10.0.0.0/16)
├── Public Subnets (for EC2, NAT Gateway)
│   ├── Public1: 10.0.1.0/24 (ap-south-1a)
│   └── Public2: 10.0.2.0/24 (ap-south-1b)
│
└── Private Subnets (for RDS)
    ├── Private1: 10.0.3.0/24 (ap-south-1a)
    └── Private2: 10.0.4.0/24 (ap-south-1b)
```
**
**Services**
| Component | Service | Description |
|------------|----------|-------------|
| Compute | EC2 (Ubuntu 24.04) | Hosts Django app with Nginx & Gunicorn |
| Database | RDS (MySQL 8.0) | Stores application data securely inside private subnet |
| Storage | EBS + S3 (optional) | Persistent storage for media/static files |
| Networking | Route 53 + Elastic IP | Custom domain integration & static IP binding |
| Security | Security Groups | Fine-grained inbound/outbound control |
| Monitoring | CloudWatch | Centralized log + performance monitoring (planned) |
| SSL | Let’s Encrypt | Free HTTPS certificate using Certbot |
**
---

## 🧩 Detailed Setup Steps

### 1️⃣ VPC & Networking
- Created **custom VPC** `blog-vpc` with 4 subnets (2 public + 2 private).  
- Configured **Internet Gateway**, **Route Tables**, and **NAT Gateway** for outbound access.
- Associated proper subnets with route tables:
  - Public subnets → Internet Gateway  
  - Private subnets → NAT Gateway

---

### 2️⃣ EC2 Configuration
- Launched an **Ubuntu 24.04** instance inside the public subnet.
- Attached an **Elastic IP** (static public IP).
- Installed base packages:
  ```bash
  sudo apt update && sudo apt install -y python3-pip python3-venv nginx mysql-client
  ```
- Created virtual environment:
  ```bash
  python3 -m venv venv
  source venv/bin/activate
  ```
- Installed Django + Gunicorn:
  ```bash
  pip install django gunicorn mysqlclient
  ```
- Verified Django setup:
  ```bash
  django-admin startproject blogsite
  python manage.py runserver 0.0.0.0:8000
  ```

---

### 3️⃣ RDS Setup (MySQL)
- Created an **RDS instance** (`db.t3.micro`, Multi-AZ disabled).  
- Subnet group: `blog-db-subnet_group` (private1 + private2 subnets).  
- Allowed inbound connection from EC2’s security group only:
  ```bash
  aws ec2 authorize-security-group-ingress   --group-id <rds-sg-id>   --protocol tcp --port 3306   --source-group <ec2-sg-id>
  ```
- Verified MySQL connection:
  ```bash
  mysql -h <rds-endpoint> -u admin -p
  ```

- Updated Django `settings.py`:
  ```python
  DATABASES = {
      'default': {
          'ENGINE': 'django.db.backends.mysql',
          'NAME': 'blogdb',
          'USER': 'admin',
          'PASSWORD': '********',
          'HOST': '<rds-endpoint>',
          'PORT': '3306',
      }
  }
  ```

- Ran migrations:
  ```bash
  python manage.py migrate
  ```

---

### 4️⃣ Gunicorn Setup
Created **systemd service** at `/etc/systemd/system/gunicorn.service`:

```ini
[Unit]
Description=Gunicorn daemon for Blog Django app
After=network.target

[Service]
User=www-data
Group=www-data
WorkingDirectory=/srv/blog-app
ExecStart=/srv/blog-app/venv/bin/gunicorn --workers 3 --bind unix:/srv/blog-app/blog.sock blogsite.wsgi:application

[Install]
WantedBy=multi-user.target
```

Commands:
```bash
sudo systemctl daemon-reload
sudo systemctl enable gunicorn
sudo systemctl start gunicorn
sudo systemctl status gunicorn
```

---

### 5️⃣ Nginx Configuration
Nginx reverse proxy config (`/etc/nginx/sites-available/prolynknetwork.in`):

```nginx
server {
    listen 80;
    server_name prolynknetwork.in www.prolynknetwork.in;

    location = /favicon.ico { access_log off; log_not_found off; }
    location /static/ {
        root /srv/blog-app;
    }

    location / {
        include proxy_params;
        proxy_pass http://unix:/srv/blog-app/blog.sock;
    }
}
```

Enable and reload:
```bash
sudo ln -s /etc/nginx/sites-available/prolynknetwork.in /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx
```

---

### 6️⃣ Domain Configuration (Route 53 + GoDaddy)
- Domain: `prolynknetwork.in` (managed on GoDaddy)
- Updated A record to point to EC2 Elastic IP.
- Added subdomains:
  ```
  prolynknetwork.in     → 13.x.x.x
  www.prolynknetwork.in → 13.x.x.x
  blog.prolynknetwork.in → 13.x.x.x
  ```

---

### 7️⃣ SSL Setup (Let’s Encrypt)
Installed and configured HTTPS:
```bash
sudo apt install certbot python3-certbot-nginx -y
sudo certbot --nginx -d prolynknetwork.in -d www.prolynknetwork.in -d blog.prolynknetwork.in
```
Verified:
```bash
sudo certbot renew --dry-run
```

---

### 8️⃣ Testing & Validation
- Access via `http://prolynknetwork.in` → Redirects to HTTPS  
- Verified Gunicorn + Nginx logs:
  ```bash
  sudo journalctl -u gunicorn --no-pager | tail
  sudo tail -f /var/log/nginx/error.log
  ```
- Final site accessible globally with valid SSL.

---

## 🔒 Security Configuration
- EC2 SG:
  - `22/tcp` → Admin IP only
  - `80,443/tcp` → Open to all
- RDS SG:
  - `3306/tcp` → EC2 SG only
- IAM Policy for EC2:
  - `AmazonRDSFullAccess`, `AmazonS3ReadOnlyAccess`

---

## 🧠 Future Enhancements
- Add **CloudWatch agent** for central logging.
- Automate **daily RDS backups** + AMI snapshots.
- Deploy static assets to **S3 + CloudFront**.
- Add **CI/CD pipeline** via GitHub Actions.
- Dockerize app for container deployment on ECS/EKS.

---

## 📦 Technologies Used
- **AWS Services:** EC2, RDS, Route53, IAM, VPC, Elastic IP, CloudWatch  
- **Web Stack:** Django, Gunicorn, Nginx, MySQL  
- **Security:** IAM, Security Groups, Let’s Encrypt SSL  
- **OS:** Ubuntu 24.04 LTS  
- **Language:** Python 3.12

---

## 🧾 Project Outcome
✅ Successfully deployed a **production-grade Django web app** on AWS with:
- Custom VPC design (segregated public/private subnets)
- Secure RDS integration
- Reverse proxy via Nginx + Gunicorn
- Domain + SSL using Let’s Encrypt
- Fully accessible via public domain:  
  👉 [https://prolynknetwork.in](https://prolynknetwork.in)

---

## 👨‍💻 Author
**Sahil — IT Admin & Cloud Engineer in transition to AWS Domain**  
_This project demonstrates practical AWS, Linux, and deployment skills for real-world cloud architecture._
**
