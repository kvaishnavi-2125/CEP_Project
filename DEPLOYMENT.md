# AWS Deployment Guide: Fern Helper

## Overview
- **Frontend**: React + Vite → **AWS S3 + CloudFront**
- **Backend**: Express.js → **AWS EC2**
- **Database**: Supabase (cloud-hosted)

---

## PHASE 1: Prerequisites

### 1. Create AWS Account
- Go to: https://aws.amazon.com
- Sign up with your email
- Add payment method

### 2. Set up AWS Credentials
```bash
# Install AWS CLI
# Download from: https://aws.amazon.com/cli/

# Configure credentials
aws configure
# Enter:
# - AWS Access Key ID
# - AWS Secret Access Key
# - Default region: us-east-1
# - Default output format: json
```

### 3. Create S3 Bucket (for Frontend)
```bash
# Replace 'fern-helper-bucket' with your unique bucket name
aws s3 mb s3://fern-helper-bucket --region us-east-1
```

---

## PHASE 2: Deploy Backend to EC2

### Step 1: Launch EC2 Instance
1. Go to **AWS Console** → **EC2**
2. Click **Launch Instance**
3. Configure:
   - **AMI**: Ubuntu 22.04 LTS (Free Tier eligible)
   - **Instance Type**: t2.micro (Free Tier)
   - **Key Pair**: Create new → Save `.pem` file safely
   - **Security Group**: 
     - SSH: port 22 (from your IP)
     - HTTP: port 80 (from anywhere)
     - Custom: port 8787 (from anywhere)
   - **Storage**: 20 GB (Free Tier)
4. Click **Launch**

### Step 2: Connect to EC2
```bash
# Connect via SSH (on Windows, use PowerShell or Git Bash)
ssh -i "your-key-pair.pem" ubuntu@your-ec2-public-ip

# Once connected, run:
sudo apt update && sudo apt upgrade -y

# Install Node.js
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt install -y nodejs npm

# Verify installation
node --version
npm --version
```

### Step 3: Deploy Backend Code
```bash
# Clone your repository (or upload files via SCP)
# Option 1: Clone from GitHub
git clone https://github.com/your-username/your-repo.git
cd cep-backend

# Install dependencies
npm install

# Create .env file with production values
nano .env
# Add:
# SUPABASE_URL=https://your-supabase-url.supabase.co
# SUPABASE_ANON_KEY=your-key
# GOOGLE_API_KEY=your-key
# RESEND_API_KEY=your-key
# FRONTEND_URL=https://your-cloudfront-url.cloudfront.net
# PORT=8787
# NODE_ENV=production

# Build TypeScript
npm run build

# Install PM2 (process manager)
sudo npm install -g pm2

# Start application with PM2
pm2 start dist/index.js --name "fern-helper-backend"
pm2 startup
pm2 save

# Verify it's running
pm2 status
```

### Step 4: Set up Nginx (Reverse Proxy)
```bash
# Install Nginx
sudo apt install -y nginx

# Create Nginx config
sudo nano /etc/nginx/sites-available/default

# Replace with:
server {
    listen 80 default_server;
    listen [::]:80 default_server;
    
    server_name _;
    
    location / {
        proxy_pass http://localhost:8787;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}

# Save and exit (Ctrl+X, then Y, then Enter)

# Test Nginx config
sudo nginx -t

# Start Nginx
sudo systemctl start nginx
sudo systemctl enable nginx
```

### Step 5: Get Elastic IP (Optional but recommended)
```bash
# Go to EC2 Console → Elastic IPs
# Allocate new address
# Associate with your instance
# Update FRONTEND_URL in backend .env
```

---

## PHASE 3: Deploy Frontend to S3 + CloudFront

### Step 1: Update Frontend Configuration
Edit [fern-helper-main/.env](fern-helper-main/.env):
```
VITE_SUPABASE_URL=https://your-supabase-url.supabase.co
VITE_SUPABASE_ANON_KEY=your-anon-key
VITE_APP_BACKEND_BASE_URL=http://your-ec2-public-ip:8787
```

### Step 2: Build Frontend
```bash
cd fern-helper-main

# Install dependencies
npm install

# Build for production
npm run build

# This creates a 'dist' folder
```

### Step 3: Upload to S3
```bash
# Upload all files from dist/ to S3 bucket
aws s3 sync dist/ s3://fern-helper-bucket/ --delete

# Make files public (S3 bucket policy)
# Go to AWS Console → S3 → Your Bucket → Permissions
# Add Bucket Policy:
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "PublicReadGetObject",
            "Effect": "Allow",
            "Principal": "*",
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::fern-helper-bucket/*"
        }
    ]
}
```

### Step 4: Create CloudFront Distribution
1. Go to **AWS Console** → **CloudFront**
2. Click **Create Distribution**
3. Configure:
   - **Origin**: Select your S3 bucket
   - **Viewer Protocol Policy**: Redirect HTTP to HTTPS
   - **Allowed HTTP Methods**: GET, HEAD, OPTIONS
   - **Cache Policy**: Managed-CachingOptimized
   - **Default Root Object**: index.html
4. Create
5. Wait ~5-10 minutes for deployment
6. Get your **CloudFront domain** (e.g., d1234.cloudfront.net)

### Step 5: Update Backend CORS
Edit `cep-backend/src/index.ts`:
```typescript
app.use(cors({
  origin: "https://your-cloudfront-domain.cloudfront.net",
  methods: ["GET", "POST", "PUT", "DELETE"],
  allowedHeaders: ["Content-Type"],
  credentials: true,
}));
```

Redeploy backend:
```bash
npm run build
pm2 restart fern-helper-backend
```

---

## PHASE 4: Setup Custom Domain (Optional)

### Using Route 53
1. Go to **AWS Console** → **Route 53**
2. Create hosted zone (if you own domain)
3. Add A records pointing to CloudFront
4. Update your domain registrar nameservers

---

## PHASE 5: SSL Certificate (HTTPS)

### AWS Certificate Manager (Free)
1. Go to **AWS Console** → **Certificate Manager**
2. Request public certificate
3. Add domain names
4. Validate via DNS
5. Attach to CloudFront distribution

---

## PHASE 6: Monitoring & Maintenance

### Monitor EC2
```bash
# Check logs
pm2 logs fern-helper-backend

# Monitor resource usage
htop  # Install: sudo apt install htop

# Check disk space
df -h

# Auto-restart on server reboot
pm2 startup
pm2 save
```

### Monitor S3 & CloudFront
- AWS Console → CloudFront → Monitoring tab
- Track requests, errors, cache hits

### Auto-scaling (Future)
- Set up Application Load Balancer
- Create EC2 auto-scaling group
- Configure target groups

---

## COST ESTIMATION (Free Tier)

✅ **Free for 12 months:**
- EC2: t2.micro (750 hours/month)
- S3: 5 GB storage + 20,000 GET requests
- CloudFront: 1 TB data transfer
- Route 53: Hosted zone ($0.50/month)

**Approx costs after free tier:**
- EC2 t2.micro: ~$8/month
- S3 storage: ~$0.023/GB
- CloudFront: ~$0.085/GB
- **Total: ~$10-15/month** (very basic usage)

---

## TROUBLESHOOTING

### Backend won't start
```bash
# Check ports
sudo netstat -tuln | grep 8787

# Check Node installation
node --version

# Check logs
pm2 logs
```

### Frontend can't reach backend
```bash
# Check CORS settings
# Check backend URL in .env
# Check EC2 security group (port 8787 open?)
```

### S3 files not updating
```bash
# Clear CloudFront cache
aws cloudfront create-invalidation \
  --distribution-id YOUR_DISTRIBUTION_ID \
  --paths "/*"
```

---

## NEXT STEPS

1. ✅ Set up AWS account
2. ✅ Deploy backend to EC2
3. ✅ Deploy frontend to S3 + CloudFront
4. ✅ Test everything
5. ⭐ Scale up as needed (RDS for database, Lambda for background jobs, etc.)

---

## PER-PHASE COMMAND CHEAT SHEET

```bash
# Backend updates
cd cep-backend
npm run build
pm2 restart fern-helper-backend

# Frontend updates
cd fern-helper-main
npm run build
aws s3 sync dist/ s3://fern-helper-bucket/
# Clear CF cache (see above)

# Check status
pm2 status
aws s3 ls s3://fern-helper-bucket/
```

