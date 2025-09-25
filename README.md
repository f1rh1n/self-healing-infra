Self-Healing Infrastructure (AWS + Terraform)

Production-style, fault-tolerant web stack on AWS. If an instance dies, the Auto Scaling Group (ASG) replaces it, traffic stays healthy behind an Application Load Balancer (ALB), and health checks gate rotation. Reproducible with Terraform.

TL;DR
# 1) Configure AWS creds (env vars or ~/.aws)
# 2) Fill in minimal variables
cp terraform.tfvars.example terraform.tfvars

# 3) Deploy
terraform init
terraform apply -auto-approve

# 4) Grab the ALB DNS and hit it
terraform output alb_dns_name

# 5) Chaos test: kill an instance and watch it heal
aws autoscaling terminate-instance-in-auto-scaling-group \
  --instance-id $(aws ec2 describe-instances --filters "Name=tag:Name,Values=self-healing-ec2" \
  --query "Reservations[0].Instances[0].InstanceId" --output text) \
  --should-decrement-desired-capacity false

Architecture

ALB (HTTP/HTTPS) with target group health checks

ASG across 2+ AZs, rolling updates, instance warm-up

EC2 (Amazon Linux 2023) with a simple web app + systemd

CloudWatch metrics/alarms (HTTP 5xx, UnHealthyHostCount)

IAM least-privileged instance profile

Optional: SSM Agent for shell-less ops

flowchart LR
  U[Users] -->|HTTPS| ALB[Application Load Balancer]
  ALB --> TG[Target Group (Health checks)]
  TG --> ASG[Auto Scaling Group]
  ASG --> EC2A[(EC2 AZ-A)]
  ASG --> EC2B[(EC2 AZ-B)]
  subgraph Observability
    CW[CloudWatch Metrics/Alarms]
  end
  ALB --> CW
  ASG --> CW
  EC2A --> CW
  EC2B --> CW

Features

Self-healing via ASG + health checks

Zero-touch rolling replacement

IaC: Terraform, idempotent, tagged, outputs wired

Observability: CloudWatch dashboards + alarms

Security: SG least-open, IAM scoped, SSM optional

Stack

AWS: ALB, ASG, EC2, CloudWatch, IAM, SSM (opt)
Terraform: v1.6+ (AWS provider v5+)
OS/App: Amazon Linux 2023, nginx (default) or your app

Repo Layout
.
├─ main.tf
├─ variables.tf
├─ outputs.tf
├─ modules/
│  ├─ networking/      # VPC, subnets, routes, SGs
│  ├─ compute/         # Launch template, ASG, userdata
│  └─ alb/             # ALB, target group, listeners
├─ userdata/
│  └─ web.sh           # Installs nginx or your app + health endpoint
├─ dashboards/
│  └─ cloudwatch.json  # Optional dashboard
├─ .github/workflows/
│  └─ terraform.yml    # Plan on PR, apply on main (optional)
├─ Makefile
└─ README.md

Quick Start
Prereqs

AWS account + credentials (AWS_PROFILE or env vars)

Terraform ≥ 1.6

(Optional) make, AWS CLI v2

Variables

terraform.tfvars.example

project         = "self-healing-infra"
region          = "ap-south-1"
vpc_cidr        = "10.50.0.0/16"
public_subnets  = ["10.50.1.0/24","10.50.2.0/24"]
instance_type   = "t3.micro"
min_size        = 2
max_size        = 4
desired_capacity= 2
health_path     = "/healthz"
health_code     = "200"
enable_https    = false     # true to attach ACM cert + 443 listener


Key inputs (see variables.tf):

region, instance_type

ASG sizes: min_size, max_size, desired_capacity

health_path, health_code for ALB checks

Deploy
terraform init
terraform validate
terraform plan
terraform apply -auto-approve

Outputs
terraform output
# alb_dns_name   = "self-healing-alb-123.ap-south-1.elb.amazonaws.com"
# target_group_arn, asg_name, vpc_id, ...


Hit the app:

curl http://$(terraform output -raw alb_dns_name)/healthz

What Makes It “Self-Healing”?

ALB → Target group health checks mark an instance unhealthy.

ASG termination policy kills unhealthy nodes.

Desired capacity maintained: ASG launches a fresh instance (from Launch Template).

Warm-up + ELB deregistration delay prevent flapping.

Failure Drills (Do These)

Kill an instance

INSTANCE_ID=$(aws ec2 describe-instances \
  --filters "Name=tag:Project,Values=self-healing-infra" "Name=instance-state-name,Values=running" \
  --query "Reservations[].Instances[].InstanceId" --output text | awk '{print $1}')
aws autoscaling terminate-instance-in-auto-scaling-group \
  --instance-id $INSTANCE_ID --should-decrement-desired-capacity false


Expected: target count dips, ASG launches replacement, ALB healthy hosts return to steady state.

Break the app

# SSH or SSM in, stop nginx:
sudo systemctl stop nginx


Expected: ALB marks instance unhealthy → ASG replaces.

Observability

CloudWatch Alarms: ALB 5xx, UnHealthyHostCount > 0, TargetResponseTime p90

Dashboard: import dashboards/cloudwatch.json

Logs: /var/log/nginx/* + /var/log/cloud-init.log

Security Posture

SGs: ALB 80/443 from 0.0.0.0/0; EC2 only from ALB SG.

IAM: instance profile limited to CloudWatch:PutMetricData, SSM (if enabled). No wildcards.

OS hardening: non-root service user, auto-updates on boot.

Secrets: none stored on instance; use SSM Parameter Store if needed.

Minimal instance policy:

{
  "Version":"2012-10-17",
  "Statement":[
    {"Effect":"Allow","Action":["logs:CreateLogGroup","logs:CreateLogStream","logs:PutLogEvents"],"Resource":"*"},
    {"Effect":"Allow","Action":["ssm:DescribeAssociation","ssm:GetDocument","ssm:ListAssociations","ssm:UpdateAssociationStatus","ssm:UpdateInstanceInformation"],"Resource":"*"}
  ]
}

User Data (health endpoint)

userdata/web.sh

#!/usr/bin/env bash
set -euo pipefail

dnf -y update
dnf -y install nginx
cat >/usr/share/nginx/html/healthz <<'EOF'
ok
EOF
cat >/etc/nginx/conf.d/default.conf <<'EOF'
server {
  listen 80 default_server;
  location / { return 200 "hello from self-healing infra\n"; }
  location /healthz { return 200 "ok\n"; }
}
EOF
systemctl enable --now nginx

Makefile (quality of life)
init:
\tterraform init

plan:
\tterraform plan

apply:
\tterraform apply -auto-approve

destroy:
\tterraform destroy -auto-approve

url:
\t@terraform output -raw alb_dns_name

CI (optional) – GitHub Actions

.github/workflows/terraform.yml

name: terraform
on:
  pull_request:
    paths: ["**.tf", "userdata/**", ".github/workflows/terraform.yml"]
  push:
    branches: ["main"]
jobs:
  plan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3
        with: { terraform_version: 1.7.5 }
      - run: terraform init
      - run: terraform fmt -check
      - run: terraform validate
      - run: terraform plan -no-color

Cost

Free tier friendly with t3.micro, ALB will cost a bit. Expect a few dollars/month. Destroy when done:

terraform destroy -auto-approve

Troubleshooting

ALB 5xx: check target health → instance systemctl status nginx → userdata logs.

No replacement: verify ASG health check type = ELB, not EC2.

Stuck in Initial: target registration delay too high; lower or ensure app responds on /healthz.

Roadmap

HTTPS via ACM + 443 listener

Blue/Green via two target groups

App autoscaling on ALB request count

SSM Session Manager only (no SSH)

License

MIT — use it, ship it.
