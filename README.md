# Travel Memory (MERN Stack) – AWS EC2 Deployment

This document describes the complete process to deploy the **Travel Memory MERN stack application** on **Amazon EC2**, configure **NGINX as a reverse proxy**, enable **scalability using Application Load Balancer and Auto Scaling**, and connect a **custom domain using GoDaddy DNS**.

Original Source Repository:  
https://github.com/UnpredictablePrashant/TravelMemory

---

## Architecture Overview

```
User Browser
     |
     v
Custom Domain (GoDaddy DNS)
     |
     v
Application Load Balancer (ALB)
     |
     v
Auto Scaling Group (Multiple EC2 Instances)
     |
     +-- NGINX (Port 80)
           |
           +-- React Frontend (/)
           +-- Reverse Proxy (/api)
                 |
                 v
             Node.js Backend (PM2, Port 3000)
                 |
                 v
             MongoDB Atlas
```

---

## Prerequisites

- AWS account
- Ubuntu 22.04 LTS EC2 instance
- MongoDB Atlas database
- GitHub account
- GoDaddy domain (for domain setup)

---

## Step 1: EC2 Instance Setup

- Launch an **Ubuntu 22.04 LTS** EC2 instance
- Instance type: `t2.micro`
- Configure Security Group inbound rules:
  - SSH (22) – Your IP only
  - HTTP (80) – 0.0.0.0/0

Connect to the EC2 instance using SSH.

---

## Step 2: Install Required Software

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y nginx git curl
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs
sudo npm install -g pm2
```

Verify installation:
```bash
node -v
npm -v
nginx -v
pm2 -v
```

---

## Step 3: Clone the Project Repository

```bash
git clone https://github.com/UnpredictablePrashant/TravelMemory.git
cd TravelMemory
```

---

## Step 4: Backend Configuration (Node.js)

Navigate to backend directory and install dependencies:

```bash
cd backend
npm install
```

Create `.env` file **inside backend directory only**:

```env
PORT=3000
MONGO_URL=mongodb+srv://<username>:<password>@cluster0.mongodb.net/travelmemory
JWT_SECRET=secure_secret
NODE_ENV=production
```

Start backend using PM2:

```bash
pm2 start index.js --name backend
pm2 save
pm2 startup
```

Verify backend:

```bash
curl http://localhost:3000
pm2 list
```

Note: `Cannot GET /` is expected because this is an API-only backend.

---

## Step 5: Frontend Configuration (React)

Navigate to frontend directory and install dependencies:

```bash
cd ../frontend
npm install
```

Update backend API path in `src/urls.js`:

```js
export const baseUrl = "/api";
```

Build the React application:

```bash
npm run build
```

---

## Step 6: NGINX Reverse Proxy Configuration

Deploy frontend build:

```bash
sudo rm -rf /var/www/html/*
sudo cp -r build/* /var/www/html/
```

Create NGINX configuration:

```bash
sudo nano /etc/nginx/sites-available/travelmemory
```

Paste the following:

```nginx
server {
    listen 80 default_server;
    server_name _;

    root /var/www/html;
    index index.html;

    location / {
        try_files $uri /index.html;
    }

    location /api {
        proxy_pass http://127.0.0.1:3000;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

Enable configuration and restart NGINX:

```bash
sudo ln -s /etc/nginx/sites-available/travelmemory /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx
```

Verify:

```bash
curl http://localhost
curl http://localhost/api
```

---

## Step 7: Scaling the Application

### 7.1 Create Amazon Machine Image (AMI)

- EC2 → Instances → Select running instance
- Actions → Image and templates → Create image
- Name the image: `travelmemory-ami`
- Wait until AMI status becomes **Available**

### 7.2 Create Target Group

- Target type: Instance
- Protocol: HTTP
- Port: 80
- Health check path: `/`

### 7.3 Create Application Load Balancer (ALB)

- Type: Application Load Balancer
- Scheme: Internet-facing
- Listener: HTTP (80)
- Select at least two subnets
- Attach the target group
- Note the **ALB DNS name**

### 7.4 Create Auto Scaling Group

- Launch template using `travelmemory-ami`
- Desired capacity: 2
- Minimum capacity: 2
- Maximum capacity: 4
- Attach to Application Load Balancer

Verify:
- Target group shows instances as **Healthy**
- Application accessible via ALB DNS

---

## Step 8: Domain Setup Using GoDaddy

Add the following DNS records in GoDaddy:

**CNAME Record**
- Host: `www`
- Points to: `<ALB-DNS-NAME>`
- TTL: 600

**A Record**
- Host: `@`
- Points to: `<EC2-PUBLIC-IP>`
- TTL: 600

Wait 5–30 minutes for DNS propagation.

Verify:
```
http://www.yourdomain.com
```

---

## Security Best Practices

- Backend port 3000 is not exposed publicly
- Secrets stored only in `.env`
- PM2 used for process management
- NGINX reverse proxy enabled
- Least-privilege security groups applied
- Health checks enabled on ALB

---

## Verification Checklist

- EC2 instance running
- PM2 backend online
- NGINX active
- Application loads using EC2 IP
- Application loads using ALB DNS
- Application loads using custom domain

---

## Submission Details

- Code pushed to GitHub (excluding `.env` and `node_modules`)
- README.md included
- Deployment PDF and architecture diagram submitted via Vlearn

---

## Conclusion

The Travel Memory MERN stack application has been successfully deployed on AWS EC2 with a scalable, secure, and production-ready architecture using NGINX, PM2, Application Load Balancer, Auto Scaling Group, and custom domain integration.
