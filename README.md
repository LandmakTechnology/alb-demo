# Landmark Technologies — ALB & Auto Scaling Demo

Deploy the Landmark Technologies web page on EC2 instances behind an Application Load Balancer with Auto Scaling and a custom domain name.

---

## What We Are Trying to Achieve

In a real production environment, you never run a single server. If that server dies, your application goes down. Instead, we set up:

1. **Multiple EC2 instances** running the same application — so if one dies, others handle traffic
2. **An Application Load Balancer (ALB)** — sits in front of all instances and distributes traffic evenly. Users hit ONE URL and the ALB decides which server handles the request
3. **An Auto Scaling Group (ASG)** — automatically adds more servers when traffic is high and removes them when traffic is low. Also replaces crashed servers automatically
4. **A custom domain name** — so users access `www.mylandmarktech.com` instead of an ugly AWS DNS name

### The Problem We Are Solving

```
Without ALB + ASG:                    With ALB + ASG:

┌────────┐     ┌──────┐              ┌────────┐     ┌─────┐     ┌──────┐
│  User  │────▶│ EC2  │              │  User  │────▶│ ALB │────▶│ EC2  │
└────────┘     └──────┘              └────────┘     │     │     ├──────┤
                                                    │     │────▶│ EC2  │
• Single point of failure                           │     │     ├──────┤
• Can't handle traffic spikes                       │     │────▶│ EC2  │
• If server dies = app is DOWN                      └─────┘     └──────┘
                                                    
                                      • High availability
                                      • Auto-scales with traffic
                                      • Self-heals (replaces dead servers)
                                      • Users see ONE URL
```

### How the Components Work Together

| Component | Role | Analogy |
|-----------|------|--------|
| **Launch Template** | Blueprint for each server (what AMI, instance type, startup script) | A recipe for baking identical cakes |
| **Auto Scaling Group** | Manages the fleet of servers (launches, monitors, replaces) | A manager who hires/fires workers based on workload |
| **Target Group** | Registry of healthy servers the ALB can send traffic to | A list of available workers |
| **ALB** | Receives all traffic and distributes across healthy servers | A receptionist directing customers to available agents |
| **Route 53** | Maps your domain name to the ALB | A phone book entry for your business |

### Flow of a User Request

```
User types: www.mylandmarktech.com
        │
        ▼
Route 53 resolves domain → ALB IP address
        │
        ▼
ALB receives request → checks Target Group for healthy instances
        │
        ▼
ALB forwards to one healthy EC2 instance (round-robin)
        │
        ▼
EC2 serves the Landmark Technologies web page
        │
        ▼
User sees the page (with that specific server's IP in the content)
```

When you refresh the page, the ALB may send you to a DIFFERENT server — you'll see a different Host IP, proving load balancing is working.

---

## Architecture

```
┌──────────────┐         ┌─────────────────────┐         ┌──────────────────────┐
│   Route 53   │────────▶│  Application Load   │────────▶│  Auto Scaling Group  │
│  (domain)    │         │    Balancer (ALB)    │         │                      │
└──────────────┘         └─────────────────────┘         │  ┌──────┐ ┌──────┐  │
                                                         │  │ EC2  │ │ EC2  │  │
                                                         │  │  #1  │ │  #2  │  │
                                                         │  └──────┘ └──────┘  │
                                                         └──────────────────────┘
```

---

## Prerequisites

- AWS account
- A VPC with at least 2 public subnets in different AZs
- (Optional) A registered domain name for Route 53

---

## Step 1: Create a Launch Template

The Launch Template defines what each EC2 instance looks like.

1. Go to **EC2 → Launch Templates → Create launch template**
2. Fill in:

| Field | Value |
|-------|-------|
| Template name | `landmark-lt` |
| AMI | Amazon Linux 2023 |
| Instance type | t2.micro |
| Key pair | Select your key pair |
| Security group | Create new: allow **HTTP (80)** from anywhere and **SSH (22)** from your IP |
| Advanced Details → User data | Paste the script from `alb-demo` file below |

### User Data Script (`alb-demo`):

```bash
#!/bin/bash
yum update -y
yum install -y httpd
systemctl start httpd
systemctl enable httpd

HOST_IP=$(hostname -f)
CURRENT_DATE=$(date '+%Y-%m-%d %H:%M:%S')

cat > /var/www/html/index.html <<EOF
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Landmark Technologies</title>
    <style>
        body { background-color: #1a1a2e; text-align: center; font-family: Arial; color: #fff; }
        h1 { color: #00d4ff; font-size: 36px; }
        h2 { color: #00d4ff; font-size: 22px; }
        .content { background: #16213e; margin: 40px auto; padding: 40px; border-radius: 12px; width: 80%; max-width: 700px; }
    </style>
</head>
<body>
    <div class="content">
        <h1>Welcome to Landmark Technologies</h1>
        <h2>Online DevOps & AI Training Platform</h2>
        <p><strong>Host:</strong> $HOST_IP</p>
        <p><strong>Deployed:</strong> $CURRENT_DATE</p>
        <p>Courses: DevOps Engineering | AI & Machine Learning | Cloud Computing | Kubernetes</p>
        <p>Email: info@mylandmarktech.com | Phone: +1 437 215 2483</p>
    </div>
</body>
</html>
EOF

systemctl restart httpd
```

3. Click **Create launch template**

---

## Step 2: Create a Target Group

The Target Group is where the ALB sends traffic.

1. Go to **EC2 → Target Groups → Create target group**
2. Fill in:

| Field | Value |
|-------|-------|
| Target type | Instances |
| Target group name | `landmark-tg` |
| Protocol | HTTP |
| Port | 80 |
| VPC | Your VPC |
| Health check path | `/` |
| Healthy threshold | 2 |
| Interval | 30 seconds |

3. Click **Next** → Do NOT register targets yet (ASG will do it automatically)
4. Click **Create target group**

---

## Step 3: Create an Application Load Balancer (ALB)

1. Go to **EC2 → Load Balancers → Create Load Balancer → Application Load Balancer**
2. Fill in:

| Field | Value |
|-------|-------|
| Name | `landmark-alb` |
| Scheme | Internet-facing |
| IP address type | IPv4 |
| VPC | Your VPC |
| Mappings | Select at least **2 public subnets** in different AZs |
| Security group | Create new: allow **HTTP (80)** from anywhere (and 443 if using HTTPS) |
| Listener | HTTP : 80 → Forward to `landmark-tg` |

3. Click **Create load balancer**
4. Wait for state to become **Active**

---

## Step 4: Create an Auto Scaling Group (ASG)

The ASG automatically launches and manages EC2 instances.

1. Go to **EC2 → Auto Scaling Groups → Create Auto Scaling group**
2. Fill in:

**Step 1 — Choose launch template:**
| Field | Value |
|-------|-------|
| Name | `landmark-asg` |
| Launch template | `landmark-lt` |

**Step 2 — Choose instance launch options:**
| Field | Value |
|-------|-------|
| VPC | Your VPC |
| Availability Zones | Select the same subnets as the ALB |

**Step 3 — Configure advanced options:**
| Field | Value |
|-------|-------|
| Load balancing | ✅ Attach to an existing load balancer |
| Choose target group | `landmark-tg` |
| Health check type | ELB |
| Health check grace period | 300 seconds |

**Step 4 — Configure group size and scaling:**
| Field | Value |
|-------|-------|
| Desired capacity | 2 |
| Minimum capacity | 2 |
| Maximum capacity | 4 |
| Scaling policy | Target tracking → Average CPU → 50% |

**Step 5 — Add tags:**
| Key | Value |
|-----|-------|
| Name | `landmark-web-server` |

3. Click **Create Auto Scaling group**

The ASG will launch 2 instances and register them with the Target Group automatically.

---

## Step 5: Test the Deployment

### 5.1 Check Target Group Health

1. Go to **EC2 → Target Groups → landmark-tg → Targets**
2. Wait until instances show **healthy**

### 5.2 Access via ALB DNS

1. Go to **EC2 → Load Balancers → landmark-alb**
2. Copy the **DNS name** (e.g., `landmark-alb-123456789.us-east-1.elb.amazonaws.com`)
3. Open in browser: `http://<alb-dns-name>`
4. Refresh multiple times — you should see different **Host IPs** (traffic distributed between instances)

### 5.3 Test Auto Scaling

Terminate one instance manually:
1. Go to **EC2 → Instances** → select one landmark instance → **Terminate**
2. Watch the ASG launch a replacement automatically (check ASG → Activity tab)

---

## Step 6: Set Up a Custom Domain (Route 53)

### 6.1 Register or Use an Existing Domain

**Option A: Register a new domain in Route 53**
1. Go to **Route 53 → Registered domains → Register domain**
2. Search for your domain (e.g., `mylandmarktech.com`)
3. Complete the registration (takes 10-30 minutes)

**Option B: Use an existing domain (transfer nameservers)**
1. Go to **Route 53 → Hosted zones → Create hosted zone**
2. Domain name: `mylandmarktech.com`
3. Type: Public hosted zone
4. Copy the 4 NS records from Route 53
5. Update your domain registrar's nameservers to these values

### 6.2 Create an Alias Record Pointing to the ALB

1. Go to **Route 53 → Hosted zones → your domain → Create record**
2. Fill in:

| Field | Value |
|-------|-------|
| Record name | (leave empty for root domain, or `www`) |
| Record type | A |
| Alias | ✅ Yes |
| Route traffic to | Alias to Application Load Balancer |
| Region | us-east-1 |
| Load balancer | `landmark-alb` |
| Routing policy | Simple |

3. Click **Create records**

### 6.3 Access via Domain

After DNS propagation (1-5 minutes):
```
http://mylandmarktech.com
http://www.mylandmarktech.com
```

---

## Step 7: (Optional) Add HTTPS with ACM

1. Go to **AWS Certificate Manager → Request certificate**
2. Domain: `mylandmarktech.com` and `*.mylandmarktech.com`
3. Validation: DNS validation → Create records in Route 53
4. Wait for status: **Issued**
5. Go to **ALB → Listeners → Add listener:**
   - Protocol: HTTPS : 443
   - Default action: Forward to `landmark-tg`
   - Certificate: Select your ACM certificate
6. (Optional) Edit HTTP:80 listener → Redirect to HTTPS:443

---

## Cleanup

To avoid charges, delete in this order:

```
1. Auto Scaling Group → Delete (will terminate instances)
2. Load Balancer → Delete
3. Target Group → Delete
4. Launch Template → Delete
```

---

## Troubleshooting

| Issue | Check |
|-------|-------|
| ALB returns 503 | Target group has no healthy targets → check Security Group allows port 80 |
| Instances unhealthy | Health check failing → SSH in and verify `curl localhost` works |
| Domain not resolving | Check Route 53 alias record → confirm ALB is Active |
| Can't access on port 80 | Security Group on instances must allow HTTP from ALB's security group |
| ASG not scaling | Check scaling policy → CloudWatch alarm must be configured |
