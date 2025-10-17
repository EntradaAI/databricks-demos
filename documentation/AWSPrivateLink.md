AWS PrivateLink provides private connectivity over the AWS backbone using interface VPC endpoints (ENIs) so traffic never traverses the public internet. With Databricks, you typically enable two kinds of PrivateLink:
1.	*Frontend PrivateLink*: private access from users/automation in your network to the workspace web UI and REST APIs. Often placed in a transit VPC connected to your corporate network (VPN/Direct Connect). Requires Enterprise plan.  ￼
2.	*Backend PrivateLink*: private connectivity from classic compute (customer VPC) to Databricks control plane core services (REST APIs + secure cluster connectivity relay). Also requires Enterprise plan, and your workspace must be in a customer managed VPC with Secure Cluster Connectivity (SCC).  ￼

PrivateLink reduces attack surface and helps with exfiltration controls. It adds hourly endpoint and per‑GB data processing charges, and introduces DNS and endpoint registration work you must design and automate up front. For many customers, the per‑GB rate is lower than NAT Gateway,

Table of contents
1.	What is AWS PrivateLink (in plain terms)
2.	How Databricks uses PrivateLink (front‑end vs back‑end; serverless)
3.	When to recommend vs. when to discourage (decision matrix)
4.	Cost model (AWS and Databricks), with worked examples
5.	Reference architectures (wiring patterns)
6.	Step‑by‑step setup (front‑end and back‑end)
7.	DNS design & verification
8.	Limitations and gotchas
9.	Pros and cons (quick list)
10.	Terraform/IaC pointers
11.	Operational runbook (monitoring, break‑glass, change control)
12.	FAQ & client talking points

1) AWS PrivateLink, in plain terms
*	What it is: A way to expose/consume services over private IPs inside your VPC, using interface VPC endpoints (ENIs). Your traffic stays on the AWS backbone; you don’t traverse the public internet. You pay per‑endpoint per‑AZ per hour and per GB of data processed by the endpoint.  ￼
*	Why it exists: To meet security/compliance controls (no public ingress/egress paths), lower exfiltration risk, and simplify allow‑listing (private IP targets).  ￼
*	How pricing works at a glance:
*	Interface endpoint hourly (per AZ) + data processed per GB (region‑dependent list price). Inter‑AZ data transfer for PrivateLink has been free since April 2022; cross‑region still incurs transfer charges in addition to PrivateLink processing.  ￼

2) How Databricks uses PrivateLink

A. Front‑end PrivateLink (users/automation → workspace UI & APIs)
*	Private access to workspace web app, REST API, Databricks Connect API via an interface endpoint you place in a transit VPC (often the VPC connected to on‑prem via VPN/Direct Connect). You can enforce private access by disabling public access at the workspace (see “Private access settings”). Enterprise plan required and customer‑managed VPC.  ￼
*	Unified login & SSO: If your users do not have public internet access, you must configure PrivateLink‑aware redirect URIs (accounts-pl-auth) and private DNS for login. (This is currently Private Preview.)  ￼

B. Back‑end PrivateLink (classic compute → control plane)
*	Two interface endpoints placed in the workspace VPC:
1.	Workspace (REST APIs) and 2) Secure Cluster Connectivity (SCC) relay. Enterprise plan, customer‑managed VPC, and SCC are required. You register these VPC endpoints with Databricks, reference them in a network configuration object, and attach that to the workspace.  ￼
*	Ports & signals you’ll see in rules: 443 (infra), 6666 (SCC/PrivateLink), 8443/8444 (internal control‑plane APIs, UC logging/lineage), 2443 (FIPS with compliance security profile). Legacy Hive metastore uses 3306 and does not go over PrivateLink.  ￼
*	Control‑plane connectivity is always over the cloud backbone (not the public internet) even without PrivateLink; PrivateLink provides additional isolation and endpoint‑based control.  ￼

C. Serverless → your VPC/resources
*	For serverless compute (Databricks managed), you can configure PrivateLink connections from serverless to services in your VPC via an NLB. Databricks charges networking costs for serverless connectivity (see pricing docs). Use this when your storage/services are private only.  ￼

3) When to recommend vs. when to discourage

Recommend PrivateLink when…
*	You must prevent public internet exposure of the workspace and/or control plane connectivity (e.g. regulated industries, strict exfiltration policies, vendor risk).  ￼
*	Users connect from on‑prem via VPN/Direct Connect and you can manage private DNS centrally (front‑end).  ￼
*	You need deterministic egress paths and per‑endpoint governance for control plane calls.
*	You want to reduce NAT Gateway costs on control‑plane/API traffic (PrivateLink’s per‑GB rate is typically lower).  ￼

Discourage/Delay PrivateLink when…
*	Your users are remote on the public internet with no corporate network path (VPN/Direct Connect). Front‑end PrivateLink would break access unless you maintain public access enabled = True or provide a VPN.  ￼
*	You cannot meet prerequisites: Enterprise plan, customer‑managed VPC, SCC. Also note some objects can’t be edited later (e.g. network configuration).  ￼
*	For pure dev/sandboxes with no strict network controls, the operational complexity (DNS, endpoint registration, change management) may outweigh benefits.

4) Cost model (with worked examples)

A. AWS charges you for interface endpoints (per AZ) and data processed
*	PrivateLink pricing: per endpoint hour (per AZ) + per GB processed. Check your region’s rates. Inter‑AZ data transfer for PrivateLink is $0 (since Apr 2022); cross‑region still incurs transfer charges.  ￼
*	NAT Gateway (for comparison): $ per hour per NAT + $ per GB processed (region‑specific, example doc shows $0.045/hour and $0.045/GB in some regions). S3/DynamoDB gateway endpoints are $0 (no hourly fee).  ￼

Important: Databricks PrivateLink carries workspace/API/relay traffic, not your data path to S3. For S3, use a Gateway VPC endpoint (free) or interface endpoint if you must access from on‑prem/peered networks. For STS/Kinesis, add interface endpoints if you operate without NAT.  ￼

B. Typical Databricks PrivateLink counts
*	Back‑end PL: 2 endpoints in the workspace VPC (workspace API + SCC).
*	Front‑end PL: 1 endpoint in the transit VPC (workspace UI/API for users).
*	HA/Multi‑AZ: Deploy endpoints in each AZ you need.

C. Worked examples (approximate, us‑region list prices)
1.	Small/medium: 2 AZs; 3 endpoints total (front‑end + 2 back‑end) ⇒ 6 interface endpoints
*	Endpoint hours: 6 × $0.01/h × 730 ≈ $43.80/mo
*	Data processed (500 GB/mo × $0.01/GB) ≈ $5.00
*	PrivateLink ≈ $48.80/mo
*	Equivalent single NAT (hourly + 500 GB @ sample rates): 0.045×730 + 0.045×500 ≈ $55.35/mo
→ PrivateLink is slightly cheaper here and more secure, with lower per‑GB than NAT. (Rates vary by region.)  ￼
2.	Large/regulated: 3 AZs; 3 endpoints ⇒ 9 interface endpoints; 10 TB (10,000 GB) of API/relay traffic
*	Endpoint hours: 9 × $0.01/h × 730 ≈ $65.70/mo
*	Data processed: 10,000 GB × $0.01 ≈ $100.00
*	PrivateLink ≈ $165.70/mo
*	Three NATs + 10 TB @ sample rates: 3×0.045×730 + 0.045×10,000 ≈ $548.55/mo
→ PrivateLink can reduce costs significantly when large volumes would otherwise traverse NAT.  ￼

Serverless networking costs: When serverless compute in Databricks connects privately to your resources, Databricks charges networking costs; consult the “Data transfer & connectivity” and “serverless networking cost” docs for specifics.  ￼

5) Reference architectures (common patterns)

Pattern A — End‑to‑end PrivateLink, classic compute
*	Transit VPC (connected to on‑prem) hosts front‑end interface endpoint (workspace UI/API).
*	Workspace VPC hosts the two back‑end endpoints (workspace REST + SCC).
*	Add S3 Gateway endpoint, STS and Kinesis interface endpoints to avoid NAT. (RDS is only relevant for legacy Hive metastore.)  ￼

Pattern B — Back‑end PrivateLink only
*	Keep public web access for users (ease of access) but make control‑plane traffic private. Good interim step when user VPN access isn’t standardized.  ￼

Pattern C — Serverless → VPC via PrivateLink
*	Databricks‑managed serverless compute connects to your NLB endpoints to reach private services (e.g. internal REST, databases). Plan for Databricks networking charges.  ￼

6) Step‑by‑step setup (what your team does)

Prereqs: Databricks Enterprise plan; customer‑managed VPC; SCC; sufficient AWS perms to create interface endpoints and manage DNS.  ￼

A. Back‑end PrivateLink (classic compute → control plane)
1.	Prepare VPC & security
*	Ensure VPC has DNS hostnames + DNS resolution enabled.
*	Open required egress ports; include 443, 6666, 8443, 8444, 2443 if using compliance security profile. (See docs for port rationale.)  ￼
2.	Create two interface endpoints (same region as workspace):
*	Service names are published by region (Workspace + SCC Relay). Use the Databricks table of VPC endpoint services to copy exact service names. Enable DNS name for each endpoint.  ￼
3.	Register endpoints with Databricks
*	In the account console, register each AWS VPC endpoint to create VPC endpoint registrations. (Note: registrations cannot be updated later.)  ￼
4.	Create a Databricks network configuration
*	Attach your VPC, subnets, SG, and the two back‑end endpoint registrations. Network configurations cannot be updated later
5.	Create/Update the workspace
*	Reference the new network configuration and private access settings (see below). After save, allow ~20 minutes before starting clusters (the UI shows RUNNING earlier).  ￼
6.	(Recommended) Add VPC endpoints to AWS services
*	S3 Gateway endpoint (free), STS interface endpoint, Kinesis interface endpoint across workspace subnets. This lets clusters operate NAT‑less.  ￼

B. Front‑end PrivateLink (users → workspace UI/API)
1.	Create interface endpoint in transit VPC for the Workspace (including REST API) service name (get from Databricks table).  ￼
2.	Register the endpoint in each Databricks account that will use it (reuse across accounts is supported).  ￼
3.	Create a Private Access Settings (PAS) object at the account level:
*	Choose region; set Public access enabled to False to enforce PrivateLink only front‑end access (or True to allow both public + PL).
*	Choose Private Access Level = Account (all registered VPC endpoints) or Endpoint (explicit allowlist).
*	You must attach a PAS when creating the workspace; you can update a workspace to point to a different PAS later.  ￼
4.	Update workspace to reference PAS (and the back‑end network config if using both). Observe the ~20 minute cluster start guidance after update.  ￼
5.	Configure internal DNS so the workspace URL resolves to the private IP of your interface endpoint (examples and nslookup checks in the doc). If using unified login without public internet, follow the accounts‑pl‑auth DNS and IdP redirect steps.  ￼

7) DNS design & verification
*	For frontend PrivateLink, you must ensure your workspace FQDN maps to the endpoint’s private IP from user networks (via conditional forwarders, private hosted zones, or A records). The doc shows the exact CNAME and A record mappings you may need, plus nslookup examples.  ￼
*	Endpoint service names & regions: use Databricks’ “PrivateLink VPC endpoint services” table to copy the correct per‑region service names for Workspace and SCC relay.  ￼

8) Limitations & gotchas (that commonly trip teams)
*	Plan tier: Enterprise is required for both frontend and backend PrivateLink.  ￼
*	Customer managed VPC: Required. You cannot convert a Databricks managed VPC workspace into customer managed later; plan new workspaces accordingly.  ￼
*	SCC required for backend PrivateLink (all new workspaces use SCC by default).  ￼
*	Registration immutability: VPC endpoint registrations and network configuration objects cannot be updated; build new ones to change. PAS can be updated or swapped.  ￼
*	Metastore note: Legacy Hive metastore (RDS/3306) does not use PrivateLink. Unity Catalog metastores are control plane services, but the RDS exceptions noted in docs are for the legacy HMS.  ￼
*	Users without VPN: If you disable public access, users must come through the private path (VPN/Direct Connect). Otherwise keep Public access enabled = True (IP access lists apply only to the public path, not PL).  ￼
*	Libraries / internet egress: Clusters often need PyPI/Maven/CRAN. Without NAT, you must:
*	Mirror repositories internally, use workspace files/volumes, or allow selective egress via a proxy/egress firewall; and add endpoints for STS/Kinesis/S3 as needed.  ￼
*	Region matching: Use PrivateLink endpoint services for the same region as your workspace or transit VPC per Databricks docs and table.  ￼

9) Pros and cons

Pros
*	Traffic to Databricks control plane and workspace stays private. Helps satisfy exfiltration and zero‑trust policies.  ￼
*	Often cheaper per GB than NAT Gateway; inter‑AZ charge is $0 for PrivateLink.  ￼
*	Clean allowlisting story, deterministic paths, and reusable endpoints across workspaces/accounts.  ￼

Cons
*	Requires Enterprise, customer managed VPC, SCC. Added DNS work, endpoint registration, and object immutability constraints.  ￼
*	End user access requires a private path (VPN/DX) if public access is disabled. SSO adds extra PL‑aware DNS work if users lack internet.  ￼
*	You still must address egress to package repos or add VPC endpoints (S3, STS, Kinesis) to operate NAT‑less.  ￼

10) Terraform/IaC pointers (resources to look up)
*	Databricks provider (account level) resources you’ll use:
*	`mws_vpc_endpoint` (register AWS VPCE with Databricks)
*	`mws_private_access_settings` (PAS)
*	`mws_networks` (network configuration referencing your VPC + endpoint registrations)
*	Databricks has an official guide “Provisioning Databricks on AWS with Private Link” with Terraform examples.  ￼

Tip: Combine with AWS provider resources for VPC endpoints and Route53 (private hosted zones/records) to automate DNS mapping required for frontend PL.  ￼

11) Operational runbook (what to standardize)
*	Change control:
*	Treat network configurations and VPC endpoint registrations as immutable—create new, switch workspace references, then deprecate old. Document 20‑minute stabilization before cluster use.  ￼
*	Monitoring & smoke tests:
*	Track endpoint health in AWS VPC console and Databricks workspace health; add synthetic tests for workspace FQDN via the private path (HTTP 200 on /api/2.0/clusters/list with a PAT).
*	Break‑glass access:
*	Keep a PAS variant with Public access enabled = True for emergency cutover if PrivateLink DNS or endpoints fail (with communications and rollback plan).  ￼
*	Cost watch:
*	Periodically review interface endpoint inventory (per‑AZ hour) and GB processed. Confirm S3 Gateway endpoint is used for in‑region buckets to avoid NAT/egress costs.  ￼

12) FAQ & client talking points

Q: Is PrivateLink required for all Databricks customers?
A: No. It’s recommended for regulated or zero‑internet patterns. Otherwise, the built‑in SCC already avoids public IPs on clusters; PrivateLink reduces exposure further by eliminating public paths to control plane and UI.  ￼

Q: Do my Delta/S3 data transfers go through Databricks PrivateLink?
A: No. Databricks PrivateLink covers the workspace/API and SCC relay. Your data access to S3 should use an S3 Gateway endpoint (free) or an interface endpoint when needed.  ￼

Q: What about Serverless?
A: When serverless workloads need to reach your VPC privately, configure PrivateLink via NLB; Databricks charges networking costs for those connections.  ￼

Q: Is PrivateLink available in our region?
A: Databricks publishes region specific endpoint service names; consult the “PrivateLink VPC endpoint services” table and choose the entries for your region.  ￼

Q: Can partners (dbt Cloud, Fivetran, Sigma, Immuta, etc.) connect to our Databricks privately?
A: Many vendors support PrivateLink to Databricks; check their docs and plan for their own PrivateLink endpoints and costs (often an enterprise/business‑critical feature on their side).  ￼
