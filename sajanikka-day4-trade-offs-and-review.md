# GreenChit — Trade-offs and Design Review

Day 4 — Deliverable 4

## Setup

**Two architectural options under review**

- **Option A — App Service monolith:** one ASP.NET Core 8 application on Azure App Service. The Claims API serves the React Web App, and the Notification Worker runs as a WebJob on the same App Service plan. Deployed as one unit. (This is the design in `sajanikka-day4-greenchit-design.md`.)
- **Option B — Container Apps split:** the Web App, Claims API and Notification Worker as three separate containers on Azure Container Apps, each built, deployed and scaled on its own, with images in Azure Container Registry.

Both options use the same Azure SQL database, Blob Storage, Service Bus, Entra ID, Teams and SharePoint, so only the hosting choice is being compared.

**Quality attributes weighted by team and business**

Scores run from 1 (poor) to 5 (excellent). Weights show how much each attribute matters to GreenChit right now:

| Weight | Meaning | Attributes |
|--------|---------|------------|
| ×3 | Decides the outcome | Time-to-first-deploy, Operability for a 10-person team |
| ×2 | Important | Cost (low spend), Authn/authz consistency |
| ×1 | Nice to have now, important later | Independent deploy, Future scaling |

**Why these weights:** Finance wants GreenChit working soon, and a 10-person team with mostly .NET skills will build and run it. The load is small: well under 50 claim submissions per hour, in business hours only. So speed and ease of running it matter most now; independent deployment and scaling matter only once GreenChit grows.

### Trade-off table

Each score has its own written justification, so every one of the 12 scored cells is explained.

| Quality attribute | Weight | Option A: App Service monolith | Why A scores this | Option B: Container Apps split | Why B scores this |
|-------------------|--------|-------------------------------|-------------------|-------------------------------|-------------------|
| Time-to-first-deploy | ×3 | **5** | One app and one CI/CD pipeline; deploying is "publish to App Service". No Dockerfiles, registry or networking to set up. The team can deploy in the first week. | **2** | Needs three Dockerfiles, Azure Container Registry, infrastructure as code for the Container Apps environment, internal networking between services, and three pipelines. Likely 2–3 extra weeks for a team new to containers. |
| Cost (low spend) | ×2 | **5** | One Premium v3 P1v3 plan with 2 instances runs everything, including the worker. No registry or extra environment to pay for. | **2** | Three services, each needing at least 2 replicas to stay available in business hours, plus Container Registry and the Container Apps environment. Paying for capacity that a load under 50 submissions an hour does not need. |
| Operability for 10-person team | ×3 | **4** | One application to monitor, one set of logs, one place to restart; the team already knows App Service. Loses a point because a worker problem can affect the API, as both share the same plan. | **3** | Clearer boundaries: a failing worker cannot slow the API. But there are three services to monitor, trace across and keep healthy, and container debugging is a new skill for this team. |
| Independent deploy | ×1 | **1** | Any change, even one line in the Teams notification code, redeploys the whole application. A bad worker release can break claim submission. | **5** | Each service is released on its own. The worker can change without touching the API, and a bad release affects only one service. |
| Future scaling | ×1 | **2** | Scales only as one unit: month-end export work and claim submissions share the same instances. Fine at today's load, a bottleneck if GreenChit grows a lot. | **5** | Each service scales on its own rules, for example the worker on queue length (KEDA) and the API on requests, down to the minimum outside business hours. |
| Authn/authz consistency | ×2 | **4** | One application validates every Entra ID token and applies one Auth & Access Policy component, so the rules cannot drift apart. Loses a point because the WebJob uses a separate entry point that must be configured correctly too. | **3** | Each service must validate tokens and apply access rules itself, and service-to-service calls need their own managed-identity setup. More places where a rule can be missed or drift out of step. |
| **Total (unweighted)** | | **21** | | **20** | Almost a tie: the plain total does not decide this. |
| **Total (weighted)** | | **48** | | **35** | With the business weights, Option A is clearly ahead. |

How the weighted totals are calculated:
- **A:** (5×3) + (5×2) + (4×3) + (1×1) + (2×1) + (4×2) = 15 + 10 + 12 + 1 + 2 + 8 = **48**
- **B:** (2×3) + (2×2) + (3×3) + (5×1) + (5×1) + (3×2) = 6 + 4 + 9 + 5 + 5 + 6 = **35**

## Results Summary

| Metric | Target | Achieved |
|--------|--------|----------|
| Quality attributes scored | 6 | 6 |
| Cells with a written justification | 12 | 12 (every A and B score has its own "why") |
| Decision-affecting attributes identified | 2-3 | 2 (Time-to-first-deploy, Operability for a 10-person team) |

## Decision and rationale

- **Option A, the App Service monolith, wins.** This is recorded in `adrs/0002-hosting-platform.md`.
- **The two attributes that drove the decision:**
  1. **Time-to-first-deploy (A 5, B 2).** Finance wants GreenChit working soon. Option B would spend weeks on container tooling before any claim could be submitted.
  2. **Operability for a 10-person team (A 4, B 3).** The team knows App Service and ASP.NET Core. One application is something they can monitor and fix confidently from day one.
- **We did not let the total decide.** The unweighted totals (21 vs 20) are almost equal, so taken alone they would say "it doesn't matter". It does matter: Option B's big advantages (independent deploy 5, future scaling 5) solve problems GreenChit does not have yet, while its weaknesses hit exactly what the business needs now.
- **What we are giving up, honestly:** independent releases and per-service scaling. Every change redeploys everything, and a bad worker release can affect claim submission. ADR-0002 lists this as the uncomfortable consequence.
- **What would make us switch to Option B:** traffic regularly above 200 requests per second, two teams needing to release parts of GreenChit separately, or the worker's month-end load slowing down claim submission. Any of these triggers a new ADR that supersedes ADR-0002.

## Design review feedback (received from another pair)

> **To complete during the session.** Write down what the reviewing pair told you, in their words where possible. Each point should name **which file and section** it refers to.

**Reviewed by:** _(names of the reviewing pair)_

**3 strengths**
1. _(strength, with the file or diagram it refers to)_
2. _
3. _

**3 weaknesses or risks**
1. _(a real risk or disagreement, not a compliment)_
2. _
3. _

**2 actionable improvements**
1. _(file, section, and the suggested change)_
2. _

**Our response:** _(for each improvement: accepted and changed, or not accepted and why)_

## Design review feedback (given to another pair)

> **To complete during the session.** Review the other pair's pack with the design-review checklist: does it solve the stated problem; are failure modes considered; are trade-offs explicit; could a new engineer pick it up; and where would we push back. Every review must contain at least one real disagreement.

**Pack reviewed:** _(names of the pair)_

**3 strengths**
1. _(what they did well, naming the file or diagram)_
2. _
3. _

**3 weaknesses or risks**
1. _(a real risk: for example a missing failure mode, an arrow without a protocol, an ADR with no real alternatives)_
2. _
3. _

**2 actionable improvements**
1. _(file, section, and the suggested change, for example: "`adrs/0003-database-choice.md`, Consequences: add what becomes harder about schema changes")_
2. _
