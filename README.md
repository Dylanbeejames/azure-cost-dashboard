# Azure Cost Visibility Dashboard

WATCH ME DO THE LAB HERE - https://www.loom.com/share/3587263343c3440caa0e21c5f282be19

**Dylan Bryson** — Azure Cloud Engineer, Norfolk VA
[LinkedIn](https://www.linkedin.com/in/dylan-bryson-b24952181/) · [GitHub](https://github.com/Dylanbeejames)

---

## Summary

Most small businesses move to the cloud expecting lower costs — then the bills arrive and nobody can read them. Azure's cost data is accurate but it's not actionable. Line items like `Microsoft.Compute/virtualMachines — $340` don't tell you who spun it up, why it's running, or whether next month will be worse.

This project solves that with a fully automated cost visibility and alerting system deployed in under five minutes using Terraform. It watches your Azure subscription spend in real time, sends email alerts before thresholds are crossed, logs subscription activity into a central workspace, and surfaces everything through a dashboard any stakeholder can navigate — no third-party tools, no extra logins, no agents.

**The problem it solves:** Business owners and engineers flying blind on cloud spend until the invoice lands.  
**What it replaces:** Manual cost reviews, surprise bills, and "who spun that up?" conversations.

---

## What It Does

- Monitors your Azure subscription against a configurable monthly budget
- Fires email alerts at 25% ($50), 50% ($100), and 100% ($200) thresholds before you're already over
- Routes alerts through a Logic App that formats and delivers the notification to any inbox
- Ships subscription activity logs (administrative, security, policy) into a central Log Analytics workspace
- Surfaces spend and resource data through an Azure Workbooks dashboard built on Resource Graph queries

---

## Architecture

**Alert Flow**

| Step | Service | What It Does |
|---|---|---|
| 1 | Cost Management Budget | Monitors monthly subscription spend against a $200 limit |
| 2 | Monitor Action Group | Receives the alert and fans out to two channels |
| 3a | Email (direct) | Sends a raw notification to your inbox immediately |
| 3b | Logic App Workflow | Formats the alert payload and sends a structured email |
| 4 | Gmail Inbox | Receives the formatted cost alert |

**Logging & Visibility Flow**

| Step | Service | What It Does |
|---|---|---|
| 1 | Diagnostic Setting | Pipes subscription activity logs into Log Analytics |
| 2 | Log Analytics Workspace | Stores and indexes administrative, security, and policy logs |
| 3 | Azure Workbooks | Queries Log Analytics and Resource Graph to render the dashboard |

**Resources Deployed**

| Resource | Name |
|---|---|
| Resource Group | rg-cost-dashboard-dylan |
| Cost Management Budget | budget-cost-dylan |
| Monitor Action Group | ag-cost-alerts-dylan |
| Logic App Workflow | la-cost-alert-dylan |
| Log Analytics Workspace | law-cost-dylan |
| Diagnostic Setting | diag-sub-to-law |
| Azure Workbook | Cost Visibility Dashboard |

---

## Stack

| Layer | Service |
|---|---|
| IaC | Terraform (azurerm ~> 3.0) |
| Cost monitoring | Azure Cost Management Budgets |
| Alerting | Azure Monitor Action Groups |
| Automation | Azure Logic Apps |
| Logging | Azure Log Analytics Workspace |
| Visualization | Azure Workbooks + Resource Graph |

---

## Prerequisites

- Terraform >= 1.0
- Azure CLI >= 2.50
- An Azure subscription with Cost Management Contributor rights
- Active `az login` session

---

## Setup

**Clone and enter the directory**

```bash
git clone https://github.com/Dylanbeejames/azure-cost-dashboard.git
cd azure-cost-dashboard
```

**Fill in your variables**

Edit `terraform.tfvars`:

```hcl
yourname    = "yourname"       # lowercase, no spaces — used in resource naming
location    = "East US"
alert_email = "you@email.com"
```

**Set the budget start date**

In `main.tf`, update this to the first of the current or upcoming month:

```hcl
start_date = "2026-05-01T00:00:00Z"
```

Azure requires this to be the first of a current or future month. If it's in the past, `terraform apply` will fail with `BudgetStartDateInvalid`.

**Deploy**

```bash
terraform init
terraform plan   # verify 6 resources before applying
terraform apply
```

Deploys in under five minutes. Runs for free at lab scale.

---

## Post-Deploy: Wire the Logic App

Terraform provisions the Logic App container. The workflow itself is a two-step setup in the portal.

**Add the trigger and email action**

1. Go to `la-cost-alert-[yourname]` in the portal
2. Open **Logic app designer**
3. Add trigger → search **"request"** → select **When an HTTP request is received**
4. Save — copy the HTTP POST URL that generates
5. Add action → search **Gmail** or **Office 365 Outlook** → select **Send an email**
6. Fill in To, Subject, and Body (use dynamic content → Body from the HTTP trigger)
7. Save

**Connect it to the Action Group**

1. Go to `ag-cost-alerts-[yourname]` in the portal → Edit
2. Under **Actions**, add a new row:
   - Action type: **Logic App**
   - Name: `la-webhook`
   - Selected: your Logic App
3. Save changes

Once wired up, any budget threshold breach triggers the full chain: Cost Management → Action Group → Logic App → inbox.

---

## Dashboard

In Azure Monitor → Workbooks, create a new workbook and add this Resource Graph query:

```kusto
resourcecontainers
| where type == "microsoft.resources/subscriptions/resourcegroups"
| project resourceGroup, location
```

Save it as **Cost Visibility Dashboard** in your resource group. From here you can layer in additional queries — cost by tag, spend over time, resource counts by service — whatever your reporting needs.

---

## Teardown

```bash
terraform destroy
```

Removes everything Terraform created. The Workbook was created manually in the portal and will need to be deleted there separately.

---

## Notes

- Alerts fire on **actual** spend, not forecasted. They won't trigger until real charges cross the threshold.
- At lab scale this costs essentially nothing. Log Analytics ingestion is billed per GB — a near-idle subscription generates a few MB per month at most.
- If `terraform apply` fails on the budget with `AuthorizationFailed`, your account needs the **Cost Management Contributor** role at the subscription scope.
