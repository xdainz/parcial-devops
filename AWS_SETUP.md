# AWS Environment Setup (ECS + RDS, AWS Academy Learner Lab)

This guide creates everything the GitHub Actions workflows need to deploy the app to **ECS Fargate** with an **RDS MySQL** database, using the **AWS Console (UI)**. It is written for the **AWS Academy Learner Lab** (region `us-east-1`, no IAM role creation allowed — everything uses the pre-existing `LabRole`).

## Architecture

```
                        Internet
                           |
                  Application Load Balancer  (innovatech-alb, port 80)
                   |                    |
        default rule (/)         /api/* and /health
                   |                    |
        frontend-service          backend-service
        (Fargate, nginx :80)     (Fargate, node :3001)
                                        |
                                  RDS MySQL :3306
                                  (innovatech_ops)
```

- One ECS Fargate cluster (`innovatech-cluster`) with two services.
- The ALB routes by path: everything goes to the frontend except `/api/*` and `/health`, which go to the backend. Frontend and API share one public URL, so no CORS issues and no IPs baked into images.
- GitHub Actions builds the images, pushes them to ECR, registers a new task definition revision (injecting DB credentials from GitHub secrets), and forces a new deployment.

**The names matter.** The workflows expect exactly: cluster `innovatech-cluster`, services `backend-service` and `frontend-service`, task definition families `task-backend` and `task-frontend`. Use the names as written.

---

## 0. Prerequisites

1. Start the Learner Lab and wait for the green dot.
2. Click **AWS** to open the console. Check the region selector (top right) says **N. Virginia (us-east-1)**.

## 1. Security groups

Three groups, chained so only the ALB is open to the internet. Console: **EC2 → Network & Security → Security Groups → Create security group**.

**Group 1 — for the load balancer:**
- Security group name: `innovatech-alb-sg`
- Description: `ALB ingress`
- VPC: leave the default VPC
- Inbound rules → **Add rule**: Type `HTTP`, Source `Anywhere-IPv4` (0.0.0.0/0)
- **Create security group**

**Group 2 — for the ECS tasks:**
- Name: `innovatech-ecs-sg`, Description: `ECS tasks`, default VPC
- Inbound rules:
  - Type `HTTP` (port 80), Source: `Custom` → in the search box pick `innovatech-alb-sg`
  - Type `Custom TCP`, Port `3001`, Source: `Custom` → `innovatech-alb-sg`
- **Create security group**

**Group 3 — for the database:**
- Name: `innovatech-rds-sg`, Description: `RDS MySQL`, default VPC
- Inbound rule: Type `MYSQL/Aurora` (port 3306), Source: `Custom` → `innovatech-ecs-sg`
- **Create security group**

## 2. ECR repositories

(Skip if they already exist.) Console: **ECR → Private registry → Repositories → Create repository**.

1. Repository name: `innovatech-backend` → **Create**.
2. Repeat with `innovatech-frontend`.

Open each repository and click the **copy icon next to the URI** (looks like `123456789012.dkr.ecr.us-east-1.amazonaws.com/innovatech-backend`). These are the values for the `ECR_REPO_URL_BACKEND` and `ECR_REPO_URL_FRONTEND` GitHub secrets.

## 3. RDS MySQL

Console: **RDS → Databases → Create database**.

- **Choose a database creation method**: Standard create
- **Engine**: MySQL, Engine version: any 8.0.x
- **Templates**: Free tier (this auto-selects a single-AZ `db.t3.micro`)
- **Settings**:
  - DB instance identifier: `innovatech-db`
  - Master username: `admin`
  - Credentials management: Self managed → enter a password and **write it down** (it becomes the `DB_PASSWORD` GitHub secret)
- **Instance configuration**: `db.t3.micro` (already set by the template)
- **Storage**: 20 GiB gp2/gp3, uncheck storage autoscaling
- **Connectivity**:
  - VPC: default VPC
  - Public access: **Yes** — needed only so you can load the SQL scripts from your machine in the next step; the security group still controls who can actually connect. In a non-lab environment this would stay No.
  - VPC security group: **Choose existing** → remove `default`, select `innovatech-rds-sg`
- **Additional configuration** (expand it — easy to miss):
  - Initial database name: `innovatech_ops`
  - Uncheck **Enable automated backups** (not needed for the lab)
- **Create database**, then wait until Status shows **Available** (5–10 minutes).

Click the instance and copy the **Endpoint** from the *Connectivity & security* tab (looks like `innovatech-db.xxxxx.us-east-1.rds.amazonaws.com`). That is the value for the `RDS_IP` GitHub secret — it's a hostname, not an IP, which is fine: the backend uses it as `DB_HOST`.

## 4. Load the database schema

You need a MySQL client for this one step. Easiest from your own PC with [MySQL Workbench](https://dev.mysql.com/downloads/workbench/) (or `mysql` CLI / DBeaver if you already have one).

**4.1 Temporarily allow your PC to reach the database:**
1. **EC2 → Security Groups** → select `innovatech-rds-sg` → **Edit inbound rules**.
2. **Add rule**: Type `MYSQL/Aurora`, Source: **My IP** → **Save rules**.

**4.2 Load the scripts with MySQL Workbench:**
1. New connection: Hostname = the RDS endpoint, Port `3306`, Username `admin`, password from step 3. **Test Connection** → OK → connect.
2. In the schema list on the left, double-click `innovatech_ops` so it becomes the active (bold) schema.
3. **File → Open SQL Script** → select `bd/create_table.sql` from this repo → run it (lightning bolt icon).
4. Same with `bd/insert_table.sql`.
5. Verify: run `SELECT COUNT(*) FROM inventory_items;` — should return 4.

**4.3 Close the hole again:**
- Back in `innovatech-rds-sg` → **Edit inbound rules** → delete the *My IP* rule (keep the `innovatech-ecs-sg` one) → **Save rules**.

## 5. Target groups

Console: **EC2 → Load Balancing → Target Groups → Create target group**. (Target groups before the load balancer, because the ALB wizard asks for one.)

**Target group 1 — frontend:**
- Target type: **IP addresses** (required for Fargate — *not* Instances)
- Target group name: `tg-frontend`
- Protocol/Port: `HTTP` / `80`, VPC: default
- Health checks: path `/`
- **Next** → register no targets (ECS does that automatically) → **Create target group**

**Target group 2 — backend:**
- Target type: **IP addresses**
- Name: `tg-backend`, Protocol/Port: `HTTP` / `3001`
- Health check path: `/health`
- **Next** → no targets → **Create target group**

## 6. Application Load Balancer

Console: **EC2 → Load Balancing → Load Balancers → Create load balancer → Application Load Balancer**.

- Name: `innovatech-alb`
- Scheme: Internet-facing, IP address type: IPv4
- **Network mapping**: default VPC, check **at least two** Availability Zones (e.g. `us-east-1a` and `us-east-1b`) and pick the default subnet in each
- **Security groups**: remove `default`, select `innovatech-alb-sg`
- **Listeners and routing**: HTTP : 80 → Default action: forward to `tg-frontend`
- **Create load balancer**

**Add the path rules** that send API traffic to the backend:
1. Open the load balancer → **Listeners and rules** tab → click the `HTTP:80` listener → **Manage rules → Add rule**.
2. Rule 1: Name `api`. **Add condition** → Path → `/api/*` → Next. Action: Forward to `tg-backend`. Priority: `10`. **Create**.
3. **Add rule** again. Rule 2: Name `health`, condition Path → `/health`, forward to `tg-backend`, priority `20`. **Create**.

Back on the load balancer details page, copy the **DNS name** (looks like `innovatech-alb-1234567890.us-east-1.elb.amazonaws.com`). Then `http://<that DNS name>` is:
- the value for the `FRONTEND_IP` GitHub secret (the backend uses it as its CORS origin), and
- the URL you open in the browser to use the app.

## 7. ECS cluster and log groups

**Cluster** — Console: **ECS → Clusters → Create cluster**:
- Cluster name: `innovatech-cluster`
- Infrastructure: **AWS Fargate (serverless)** (the default)
- **Create**

**Log groups** — Console: **CloudWatch → Logs → Log groups → Create log group**:
1. Name: `/ecs/task-backend` → **Create**
2. Repeat with `/ecs/task-frontend`

## 8. GitHub secrets

In the GitHub repo: **Settings → Secrets and variables → Actions → New repository secret**. Set:

| Secret | Value |
| --- | --- |
| `AWS_ACCESS_KEY_ID` | From the Learner Lab **AWS Details** panel |
| `AWS_SECRET_ACCESS_KEY` | From AWS Details |
| `AWS_SESSION_TOKEN` | From AWS Details |
| `AWS_REGION` | `us-east-1` |
| `ECR_REPO_URL_BACKEND` | Backend repository URI from step 2 |
| `ECR_REPO_URL_FRONTEND` | Frontend repository URI from step 2 |
| `RDS_IP` | RDS endpoint from step 3 |
| `DB_NAME` | `innovatech_ops` |
| `DB_USER` | `admin` |
| `DB_PASSWORD` | The password from step 3 |
| `FRONTEND_IP` | `http://<ALB DNS name>` from step 6 |

No longer used (left over from the EC2 deployment): `BACKEND_IP`, `EC2_SSH_KEY`. They can be deleted.

> **Every time you start a new Learner Lab session** the AWS credentials rotate. Update `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY` and `AWS_SESSION_TOKEN` in GitHub before deploying. Everything else stays the same.

## 9. First deploy (push images and register task definitions)

Push to the `deploy` branch, or run both workflows manually: GitHub → **Actions** → select *Backend Deploy* → **Run workflow**, same for *Frontend Deploy*.

On the first run each workflow will:
1. Build the image and push it to ECR.
2. Register the task definition (`task-backend` / `task-frontend`).
3. Print that the ECS service doesn't exist yet — **that's expected**; you create the services next.

## 10. Create the ECS services

Console: **ECS → Clusters → innovatech-cluster → Services tab → Create**.

**Backend service:**
- **Environment** (expand *Compute configuration* if collapsed): Compute options: **Launch type** → `FARGATE`
- **Deployment configuration**:
  - Application type: Service
  - Family: `task-backend`, Revision: latest
  - Service name: `backend-service`
  - Desired tasks: `1`
- **Networking**:
  - VPC: default VPC
  - Subnets: the same two subnets you used for the ALB
  - Security group: **Use an existing security group** → remove `default`, select `innovatech-ecs-sg`
  - Public IP: **Turned on** — required so the task can pull the image from ECR (the lab VPC has no NAT gateway). Inbound traffic is still blocked by the security group except from the ALB.
- **Load balancing**:
  - Load balancer type: Application Load Balancer → **Use an existing load balancer** → `innovatech-alb`
  - Listener: **Use an existing listener** → `80:HTTP`
  - Target group: **Use an existing target group** → `tg-backend` (container `backend 3001:3001` should be pre-selected)
  - Health check grace period: `60` seconds
- **Create**

**Frontend service:** repeat with Family `task-frontend`, Service name `frontend-service`, target group `tg-frontend` (container `frontend 80:80`), everything else the same.

## 11. Verify

1. **ECS → innovatech-cluster → Services**: wait until both services show **1/1 Tasks running** and deployment status *Completed* (a few minutes; refresh).
2. **EC2 → Target Groups → tg-backend / tg-frontend → Targets tab**: one target each, status **healthy**.
3. In the browser:
   - `http://<ALB DNS name>/health` → `{"status":"ok","database":"connected"}`
   - `http://<ALB DNS name>/api/items` → JSON with the seeded products
   - `http://<ALB DNS name>` → the app, with data loaded

From now on, every push to `deploy` that touches `backend/**` or `frontend/**` builds, pushes and rolls out a new deployment automatically.

## Cost control

The Learner Lab budget is limited and RDS + Fargate + ALB consume credits while running. When you are not working on it:

- **ECS**: each service → **Update service** → Desired tasks: `0` → Update. (Set back to `1` to resume.)
- **RDS**: select `innovatech-db` → **Actions → Stop temporarily**. (Start it again before deploying; note AWS auto-restarts it after 7 days.)
- The ALB cannot be stopped, only deleted.

## Troubleshooting

- **Backend task keeps stopping / restarting**: the backend exits if it can't reach MySQL on startup. Check the logs: **ECS → cluster → backend-service → Logs tab** (or CloudWatch log group `/ecs/task-backend`). Usual causes: wrong `RDS_IP`/`DB_USER`/`DB_PASSWORD` secret (fix and re-run the backend workflow), RDS stopped, or `innovatech-rds-sg` missing the inbound rule from `innovatech-ecs-sg`.
- **`503 Service Temporarily Unavailable` on the ALB**: no healthy targets yet. Check the target group's *Targets* tab; give it a minute after a deploy.
- **Workflow fails with `ExpiredToken` / `UnrecognizedClientException`**: the Learner Lab session credentials expired. Start the lab and update the three AWS secrets in GitHub.
- **Task fails with `CannotPullContainerError`**: the service was created with Public IP turned off, or the image was never pushed (run the workflow first).
- **Frontend loads but API calls fail**: check the listener rules on the ALB (`/api/*` → `tg-backend`) and that `/health` works in the browser.
