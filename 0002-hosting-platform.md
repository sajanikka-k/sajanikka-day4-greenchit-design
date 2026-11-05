# ADR 0002: Host GreenChit as a single application on Azure App Service

## Status
Accepted (date: 2026-11-04)

## Context
- **Speed matters more than flexibility right now.** Finance wants GreenChit working soon, and a first internal release is the priority. Every week spent on infrastructure is a week staff keep using paper or spreadsheets for claims.
- **The team is small and mostly .NET developers.** The 10-person team knows ASP.NET Core well, but has limited experience with containers, container registries and infrastructure as code. Learning those under deadline pressure is a real delivery risk.
- **The load is small and predictable.** GreenChit is internal: it serves BISTEC staff, with an expected peak of well under 50 claim submissions per hour, mostly on weekdays during business hours and at month end. Nothing about this load needs services that scale separately.
- **Availability is needed in business hours only.** The target is 99.9% between 08:00 and 19:00 Sri Lanka time, Monday to Friday. A single-region managed platform with a 99.95% SLA meets this, and deployments can happen outside business hours.
- **Two Azure options are allowed:** App Service or Container Apps. The trade-off table in `trade-offs/hosting-options.md` scores them almost equally: **App Service monolith 21, Container Apps split 20**. Because the totals are so close, the decision must rest on the attributes the business cares about most, not on the total.
- **What we don't yet know:** BISTEC's exact staff growth over the next two years; whether other tools will later need to call GreenChit's API; and whether finance policy requires the data to stay in a particular Azure region.

## Decision
- **We choose Azure App Service** and run GreenChit as **one ASP.NET Core 8 application**, deployed as a single unit. The Claims API serves the React Web App's static files, and the Notification Worker runs as a **continuous WebJob on the same App Service plan**.
- **The two attributes that drove this decision**, from the trade-off table:
  1. **Time to first deploy** (App Service 5, Container Apps 2): one app, one pipeline, no container images, registry or networking setup.
  2. **Operability for a 10-person team** (App Service 4, Container Apps 3): fewer moving parts to monitor and fix, using tools the team already knows.
- **Configuration:**
  - Plan: **Premium v3 P1v3**, **2 instances** minimum, so one instance can fail or restart without downtime.
  - Autoscale: add 1 instance when average CPU is above 70% for 10 minutes; maximum 4 instances.
  - **Deployment slots:** `staging` and `production`. Every release goes to staging first and is swapped into production only after smoke tests pass, outside business hours.
  - **Always On** enabled (needed for the WebJob), and App Service **Health check** on `/health`.
  - Region: **Central India**, the closest full-featured Azure region to Colombo, unless finance policy requires otherwise.
- **When we revisit this:** we write a new ADR to consider splitting into separate services if traffic regularly exceeds 200 requests per second, if two teams need to release parts of GreenChit independently, or if the worker's load starts to slow down the API.

## Consequences
- **Easier:** a working release much sooner; one CI/CD pipeline; one place to look for logs and errors; local development is just "run the app"; deployment slots give safe releases with instant rollback by swapping back.
- **Harder:** **every change redeploys everything.** A one-line fix in the Teams notification code means redeploying the whole application, including the Claims API, and a bad release of the worker can take down claim submission too. We cannot scale the API and the worker separately: a burst of month-end export work competes for the same CPU as staff submitting claims. This is the uncomfortable part of this decision, and we accept it because the load is small.
- **Different:** the team will not build container skills on this project, although containers are becoming common across BISTEC. To limit this, one engineer builds a small Container Apps proof of concept outside the critical path, so the team is ready if we revisit this ADR.

## Alternatives considered
- **Azure Container Apps, split into Web, API and Worker services (Option B):** rejected. It scores better on independent deployment (5 vs 1) and future scaling (5 vs 2), but those benefits do not matter at under 50 submissions per hour. It costs more in time and skills: Dockerfiles, a container registry, infrastructure as code, service-to-service networking, and several replicas to pay for (cost: 2 vs 5). For this team and this load, that effort outweighs the scaling benefit.
- **Azure Functions only (serverless):** rejected. Uploading and checking receipts of up to 10 MB, and the API's request-and-response style, fit poorly with function time limits and cold starts. Cold starts would also put the 1.5-second submission target at risk on the first request of the morning.
- **Azure Kubernetes Service:** rejected without detailed scoring. It needs much more operations skill than either option above, and nothing in GreenChit's requirements justifies it.
