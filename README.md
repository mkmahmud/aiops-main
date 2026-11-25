# FlashScale AIOps Lab

A hands-on lab for learning how to **spot, investigate, and fix problems in a live web app** using AWS monitoring tools.

You run a small online store. Then you **break it on purpose** with one click (make it slow, make it throw errors, make orders pile up) and watch the problem appear in AWS CloudWatch graphs, alarms, emails, and logs. Then you reset it and try the next one.

> **AIOps** means using monitoring data, automatic anomaly detection and alerts to find and fix production problems faster.

---

## What's inside

```
   Users / load test
          │
          ▼
   ┌─────────────┐     ┌──────────────┐     ┌─────────┐     ┌──────────────┐
   │   NGINX     │ ──► │  API (Node)  │ ──► │  Redis  │ ◄── │ Worker (Node)│
   │ web server  │     │ store + lab  │     │ queue   │     │ processes    │
   └─────────────┘     └──────────────┘     └─────────┘     │ orders       │
                                                            └──────────────┘
          │                    │                                   │
          └──────── all logs go to AWS CloudWatch ─────────────────┘
                     → graphs, alarms, email alerts
```

| Part | What it does |
|---|---|
| **NGINX** | Takes incoming web traffic and passes it to the API |
| **API** | The store: lists products, accepts orders, and hosts the lab control panel at `/aiops` |
| **Redis** | Holds the queue of orders waiting to be processed |
| **Worker** | Takes orders off the queue and "processes" them |
| **CloudWatch** | AWS service that collects logs and metrics, draws graphs and fires alarms |
| **SNS** | AWS service that sends you an email when an alarm fires |

All four app parts run as Docker containers on **one EC2 server**.

---

## The incidents you can trigger

Open `http://YOUR_SERVER/aiops` and click a button, or use the scripts shown further down.

| Scenario | What breaks | What you should see |
|---|---|---|
| **Latency** | Every request is slowed down by about 0.5 seconds | Response time rises, but CPU looks normal |
| **Error spike** | 1 in 5 requests fails with a `503` error | 5xx error alarm fires and you get an email |
| **Worker bottleneck** | The worker becomes very slow | The order queue grows while the website still looks fine |
| **Queue backlog** | The worker is slow and the API has some errors | Accepted orders pile up far ahead of processed ones |
| **Log anomaly** | Unusual messages start appearing in the logs | Something to find with CloudWatch Logs Insights |
| **Normal** | Everything back to healthy | Baseline for comparison |

Each scenario **switches itself off after 15 minutes**, so nothing stays broken by accident.

---

## Quick try on your own computer (no AWS needed)

You only need **Docker** installed.

```bash
git clone https://github.com/mkmahmud/aiops-main.git
cd aiops-main
docker compose up -d --build
```

Then open:

- http://localhost: the store
- http://localhost/aiops: the incident control panel
- http://localhost/stats: queue and order counts

Stop it with `docker compose down`.

> Running locally shows the app and the incident buttons. The CloudWatch graphs, alarms and emails only work when it runs on AWS (see below).

---

## Full setup on AWS

### What you need

- An AWS account
- The AWS CLI and Terraform installed on your computer
- An existing EC2 key pair (for SSH)

### Step 1: Create the server

```bash
cd infra/terraform-ec2-minimal
cp terraform.tfvars.example terraform.tfvars
# Edit terraform.tfvars: set your key pair name and your public IP
terraform init
terraform apply
```

Terraform prints the server's IP address and an SSH command.

### Step 2: Put the code on the server

```bash
ssh -i YOUR_KEY.pem ubuntu@SERVER_IP
git clone https://github.com/mkmahmud/aiops-main.git
cd aiops-main
cp .env.example .env
# Edit .env: set ALERT_EMAIL and AIOPS_ALERT_EMAIL to your email address
```

### Step 3: Install, start and connect to CloudWatch

```bash
sudo ./scripts/setup-ec2.sh          # installs Docker and the AWS CLI
exit                                 # log out, then SSH back in so Docker works without sudo
cd aiops-main
./scripts/deploy.sh                  # builds and starts the app
sudo ./scripts/setup-cloudwatch.sh   # creates log groups, metrics and the dashboard
sudo ./scripts/setup-sns-alerts.sh   # emails you at traffic levels
sudo ./scripts/setup-aiops.sh        # creates anomaly, error and queue alarms
```

**Check your inbox** and click the AWS SNS confirmation link, otherwise you won't receive alerts.

### Step 4: Give it normal traffic first

The anomaly alarms need to learn what "normal" looks like before they're useful.

```bash
sudo ./scripts/install-periodic-small-load.sh   # small load test every 15 min
```

Leave it running for a while (a few hours is ideal) before you start breaking things.

---

## Running a lab

```bash
./scripts/aiops-run-scenario.sh latency   # turn on a problem and send some traffic
./scripts/aiops-status.sh                 # what mode is active right now?
./scripts/aiops-logs-query.sh             # pull recent suspicious log lines
./scripts/aiops-reset.sh                  # back to normal
```

Scenario names: `latency`, `error-spike`, `worker-bottleneck`, `queue-backlog`, `log-anomaly`, `normal`.

**Suggested flow for each scenario:**

1. **Baseline**: look at the CloudWatch dashboard while things are healthy.
2. **Break it**: turn on one scenario.
3. **Watch**: which graph moved? Did an alarm fire? Did you get an email?
4. **Investigate**: check the logs to find the cause.
5. **Fix**: reset, and discuss what the real-world fix would be (for example, add more workers).

Recommended order: [aiops-labs/README.md](aiops-labs/README.md).

---

## Big load tests (optional)

To hit the app with thousands of virtual users, use the
[AWS Distributed Load Testing](https://aws.amazon.com/solutions/implementations/distributed-load-testing-on-aws/) solution:

```bash
./scripts/package-dlt-k6.sh --base-url http://SERVER_IP --peak-vus 5000 --order-ratio 0.05
```

Upload the zip from `dlt/dist/` in the DLT console. Details: [dlt/README.md](dlt/README.md).

---

## Useful scripts

| Script | What it does |
|---|---|
| `status.sh` | Shows running containers, health and order stats |
| `deploy.sh` | Builds and (re)starts the app |
| `cleanup.sh` | Stops the app |
| `run-small-local-load.sh` | Runs one small load test |
| `test-sns-alert.sh` | Sends a test alert email |
| `remove-aiops.sh` / `remove-sns-alerts.sh` / `remove-periodic-small-load.sh` | Undo the matching setup script |

---

## Clean up (avoid AWS charges)

```bash
# On the server
./scripts/remove-periodic-small-load.sh
./scripts/remove-aiops.sh
./scripts/remove-sns-alerts.sh
./scripts/cleanup.sh

# On your computer
cd infra/terraform-ec2-minimal
terraform destroy
```

> The default server size (`c7i.xlarge`) costs money while it runs. Destroy it when you're done.

---

## Project layout

```
app/            API, worker and NGINX config
configs/        CloudWatch agent config
infra/          Terraform to create the EC2 server
iam/            IAM permissions the server needs
scripts/        Setup, run, lab and cleanup scripts
dlt/            Load-test template for AWS Distributed Load Testing
aiops-labs/     Lab guide and recommended order
```
