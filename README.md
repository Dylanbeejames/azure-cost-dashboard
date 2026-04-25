# Azure Cost Visibility Dashboard

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

rg-cost-dashboard-[yourname]
├── azurerm_consumption_budget_subscription   → monthly budget with 3 alert thresholds
├── azurerm_monitor_action_group              → email receiver and Logic App webhook target
├── azurerm_logic_app_workflow                → formats and delivers alert email on trigger
├── azurerm_log_analytics_workspace           → central log store for subscription activity
├── azurerm_monitor_diagnostic_setting        → pipes activity logs into Log Analytics
└── Azure Workbook (portal)                   → Resource Graph query dashboard

All infrastructure is managed by Terraform. The Logic App workflow is configured in the portal designer post-deploy.

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

Clone and enter the directory

git clone https://github.com/Dylanbeejames/azure-cost-dashboard.git
cd azure-cost-dashboard

Fill in your variables in terraform.tfvars:

yourname    = "yourname"
location    = "East US"
alert_email = "you@email.com"

Set start_date in main.tf to the first of the current or upcoming month:

start_date = "2026-05-01T00:00:00Z"

Deploy:

terraform init
terraform plan
terraform apply

Deploys in under five minutes. Runs for free at lab scale.

---

## Post-Deploy: Wire the Logic App

1. Go to la-cost-alert-[yourname] in the portal
2. Open Logic app designer
3. Add trigger → search "request" → select When an HTTP request is received
4. Save — copy the HTTP POST URL
5. Add action → search Gmail or Office 365 Outlook → select Send an email
6. Fill in To, Subject, Body (dynamic content → Body from HTTP trigger)
7. Save

Then go to ag-cost-alerts-[yourname] → Edit → Actions → add Logic App → la-webhook → save.

---

## Dashboard

In Azure Monitor → Workbooks, create a new workbook and run this query:

resourcecontainers
| where type == "microsoft.resources/subscriptions/resourcegroups"
| project resourceGroup, location

Save as Cost Visibility Dashboard in your resource group.

---

## Teardown

terraform destroy

Removes everything Terraform created. Delete the Workbook manually in the portal.

---

## Notes

- Alerts fire on actual spend, not forecasted.
- Runs for free at lab scale — Log Analytics ingestion is billed per GB.
- If terraform apply fails with AuthorizationFailed on the budget, assign Cost Management Contributor at the subscription scope.
