# Azure — Hands-On Project Lab

> Deploy a secured web app on Azure App Service with Azure SQL.

## Objective

Stand up a PaaS web app with a managed database and identity-based access control.

## Prerequisites

- Azure free account
- Basic web dev

## Steps

1. Create an Azure App Service plan and deploy a sample web app.
2. Provision an Azure SQL Database and connect the app to it via a connection string in App Settings (not code).
3. Enable Azure AD authentication on the App Service.
4. Set up a firewall rule so the SQL server only accepts traffic from the app's outbound IPs.
5. Enable Application Insights and view a live request trace.
6. Configure a deployment slot for staging and swap it into production.

## Deliverable

A live app URL, a screenshot of an Application Insights trace, and proof the slot swap caused zero downtime.

## Stretch goals

- Move the connection string into Azure Key Vault and reference it via a managed identity.

---
*Part of the [EngiDock](https://www.engidock.com) course library.*
