# AWS Setup Instructions

## 1. Create AWS S3 Bucket
- Go to AWS Console → S3 → Create Bucket
- Bucket Name: `fern-helper-uploads` (must be unique globally)
- Region: `ap-south-1` (Asia Pacific - South)
- Keep defaults, click "Create"

## 2. Create IAM User for API Access
1. Go to **IAM Dashboard** → **Users** → **Create User**
2. Username: `fern-helper-backend`
3. Next → Attach Policy: Select `AmazonS3FullAccess`
4. Create User
5. Go to **Users** → Select the user → **Security Credentials** tab
6. Click **Create Access Key** → Choose "Application running outside AWS"
7. Copy these values to `.dev.vars`:
   - `AWS_ACCESS_KEY_ID`
   - `AWS_SECRET_ACCESS_KEY`

## 3. Create EC2 Instance (Optional - for backend hosting)
1. EC2 Dashboard → **Launch Instance**
2. AMI: Ubuntu 24.04 LTS (free-eligible)
3. Instance Type: `t2.micro`
4. Key Pair: Create new one, download `.pem` file
5. Security Group: Allow SSH (22), HTTP (80), HTTPS (443)
6. Storage: 30 GB (free tier limit)

## 4. Deploy Backend to EC2
SSH into instance and run:
```bash
ssh -i your-key.pem ubuntu@your-ec2-ip

# Install Node.js
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt-get install -y nodejs

# Clone your repo
git clone <your-repo-url>
cd cep-backend

# Install dependencies + AWS SDK
npm install
npm install @aws-sdk/client-s3

# Set environment variables
nano .env  # Add your AWS credentials and S3 bucket name

# Start backend (or use PM2 for persistent running)
npm run build && npm start
```
