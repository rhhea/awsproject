#  AWS Full-Stack Login & Signup App

A full-stack user authentication app deployed on AWS using **S3**, **EC2**, and **RDS (MySQL)**. Users can sign up and log in with passwords stored securely using bcrypt hashing.

---

##  Architecture

```
Browser
  │
  ├──► S3 (Static Website) ──► Serves HTML/CSS/JS frontend
  │
  └──► EC2 (Node.js API) ──────► RDS (MySQL Database)
         Port 3000                   Port 3306
```

| Service | Role |
|---|---|
| **S3** | Hosts the static frontend (HTML/CSS/JS) |
| **EC2** | Runs the Node.js/Express backend API |
| **RDS** | MySQL database storing user data |

---

##  Project Structure

```
myapp/
├── server.js       # Node.js/Express backend
├── .env            # Environment variables (DB credentials)
├── package.json    # Node dependencies
└── node_modules/   # Installed packages (auto-generated)

S3 Bucket/
└── index.html      # Frontend (login/signup form)
```

---

##  Setup Guide

### Phase 1 — RDS (Database)

#### Step 1: Create RDS Instance
1. AWS Console → RDS → **Create database** → Full configuration
2. Settings:
   - Engine: **MySQL 8.0**
   - Template: **Dev/Test**
   - Availability: **Single-AZ**
   - DB identifier: `myproject-db`
   - Master username: `admin`
   - Instance class: `db.t3.micro`
   - Storage: 20 GB
   - VPC: **Default VPC**
   - Public access: **Yes**
   - Security group: Create new → name it `rds-sg`
3. Wait ~10 mins → copy the **Endpoint URL**

#### Step 2: Configure RDS Security Group
1. EC2 → Security Groups → `rds-sg` → Edit inbound rules
2. Add rule:
   - Type: **MYSQL/Aurora** | Port: **3306** | Source: **Anywhere IPv4**
3. Save rules

---

### Phase 2 — EC2 (Backend Server)

#### Step 3: Launch EC2 Instance
1. AWS Console → EC2 → Launch instance
2. Settings:
   - Name: `myproject-server`
   - AMI: **Amazon Linux 2023**
   - Instance type: `t2.micro`
   - Key pair: Create new → `myproject-key` → download `.pem`
   - Auto-assign public IP: **Enable**
3. Security group: Create new → `ec2-sg` with these inbound rules:

| Type | Port | Source |
|---|---|---|
| SSH | 22 | Anywhere IPv4 |
| HTTP | 80 | Anywhere IPv4 |
| Custom TCP | 3000 | Anywhere IPv4 |

4. Launch → wait until **Running** → copy **Public IPv4 address**

#### Step 4: Connect to EC2
Use **EC2 Instance Connect** (browser terminal):
> EC2 → Instances → myproject-server → Connect → EC2 Instance Connect → Connect

#### Step 5: Install Node.js on EC2
```bash
sudo dnf update -y
sudo dnf install -y nodejs npm
node -v
sudo npm install -g pm2
```

#### Step 6: Create Project Folder
```bash
mkdir myapp
cd myapp
npm init -y
npm install express mysql2 bcryptjs cors dotenv
```

#### Step 7: Create `.env` File
```bash
nano .env
```
```env
DB_HOST=myproject-db.xxxxxxx.us-east-1.rds.amazonaws.com
DB_USER=admin
DB_PASS=your-rds-password
DB_NAME=myapp
PORT=3000
```
Save: `Ctrl+X` → `Y` → `Enter`

#### Step 8: Create `server.js`
```bash
nano server.js
```
```javascript
const express = require('express');
const mysql = require('mysql2/promise');
const bcrypt = require('bcryptjs');
const cors = require('cors');
require('dotenv').config();

const app = express();
app.use(express.json());
app.use(cors());

const pool = mysql.createPool({
  host: process.env.DB_HOST,
  user: process.env.DB_USER,
  password: process.env.DB_PASS,
  database: process.env.DB_NAME,
});

// Signup route
app.post('/signup', async (req, res) => {
  const { username, email, password } = req.body;
  try {
    const hash = await bcrypt.hash(password, 10);
    await pool.execute(
      'INSERT INTO users (username, email, password_hash) VALUES (?, ?, ?)',
      [username, email, hash]
    );
    res.json({ success: true, message: 'User created successfully' });
  } catch (err) {
    if (err.code === 'ER_DUP_ENTRY') {
      return res.status(409).json({ error: 'Username or email already exists' });
    }
    res.status(500).json({ error: 'Server error' });
  }
});

// Login route
app.post('/login', async (req, res) => {
  const { email, password } = req.body;
  try {
    const [rows] = await pool.execute(
      'SELECT * FROM users WHERE email = ?', [email]
    );
    if (rows.length === 0) {
      return res.status(401).json({ error: 'Invalid credentials' });
    }
    const match = await bcrypt.compare(password, rows[0].password_hash);
    if (!match) return res.status(401).json({ error: 'Invalid credentials' });
    res.json({ success: true, message: `Welcome, ${rows[0].username}!` });
  } catch (err) {
    res.status(500).json({ error: 'Server error' });
  }
});

app.listen(process.env.PORT, () => {
  console.log(`Server running on port ${process.env.PORT}`);
});
```
Save: `Ctrl+X` → `Y` → `Enter`

#### Step 9: Set Up MySQL Database
```bash
sudo dnf install -y mariadb105
mysql -h YOUR_RDS_ENDPOINT -u admin -p
```
```sql
CREATE DATABASE myapp;
USE myapp;

CREATE TABLE users (
  id INT AUTO_INCREMENT PRIMARY KEY,
  username VARCHAR(50) NOT NULL UNIQUE,
  email VARCHAR(100) NOT NULL UNIQUE,
  password_hash VARCHAR(255) NOT NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

EXIT;
```

#### Step 10: Start Server with PM2
```bash
pm2 start server.js --name myapp
pm2 startup
pm2 save
pm2 status   # Should show myapp as "online"
```

---

### Phase 3 — S3 (Frontend)

#### Step 11: Create S3 Bucket
1. AWS Console → S3 → Create bucket
2. Name: `myproject-website-yourname` (must be globally unique)
3. Region: same as EC2
4. Uncheck **Block all public access** → acknowledge
5. Object ownership: **ACLs disabled**
6. Create bucket

#### Step 12: Enable Static Website Hosting
1. Bucket → **Properties** tab → Static website hosting → Edit → **Enable**
2. Index document: `index.html` → Save

#### Step 13: Add Bucket Policy
Bucket → **Permissions** → Bucket policy → Edit:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PublicReadGetObject",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::myproject-website-yourname/*"
    }
  ]
}
```

#### Step 14: Upload `index.html`
Create locally, replacing `EC2_PUBLIC_IP` with your actual EC2 IP, then upload to S3.

Your website URL will be:
```
http://myproject-website-yourname.s3-website-us-east-1.amazonaws.com
```

---

##  Testing Checklist

| Test | Expected Result |
|---|---|
| Open S3 website URL | Login form loads |
| Sign up with new email | "User created successfully" |
| Sign up with same email again | "Username or email already exists" |
| Login with correct credentials | "Welcome, username!" |
| Login with wrong password | "Invalid credentials" |
| Run `pm2 status` on EC2 | myapp shows as "online" |

---

##  Useful Commands

```bash
pm2 status           # Check if server is running
pm2 restart myapp    # Restart the server
pm2 logs myapp       # View server logs
pm2 stop myapp       # Stop the server
```

To check registered users in the database:
```bash
mysql -h YOUR_RDS_ENDPOINT -u admin -p
```
```sql
USE myapp;
SELECT * FROM users;
```

---

##  Issues Encountered & Fixes

| Issue | Fix |
|---|---|
| SSH not working on Windows | Used EC2 Instance Connect (browser terminal) |
| EC2 and RDS in different VPCs | Enabled public access on RDS |
| Port 3000 rule showing as port 0 | Edited Custom TCP rule to port 3000 |
| rds-sg couldn't reference ec2-sg | Used Anywhere IPv4 as source instead |

---

##  Cost & Cleanup

All services used fall under **AWS Free Tier** for the first 12 months:
- `t2.micro` EC2
- `db.t3.micro` RDS
- S3 with low traffic

> Note: When done, **terminate** (not just stop) EC2, delete RDS, and empty + delete the S3 bucket to avoid charges.

---

##  Security Notes

- Passwords are hashed using **bcrypt** before storing in the database
- For production: add HTTPS via an Application Load Balancer + ACM certificate
- For production: restrict RDS security group to EC2 security group only (same VPC)
- Consider adding JWT-based session management for authenticated routes
