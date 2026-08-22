# DigitalOcean Elasticsearch Complete Guide: Self-Hosted on a Droplet or Managed OpenSearch? How to Choose, Deploy, and Secure a Cluster Without Burning Money (With Full Plan Breakdown and $200 Free Credit)

If you've ever stared at an AWS invoice after spinning up an Elasticsearch cluster, you already know the punchline. Search infrastructure has a reputation for eating budgets — sometimes quietly, sometimes spectacularly. So when people start hunting for "DigitalOcean Elasticsearch," what they're usually really asking is something messier and more honest: *Can I run a real search stack somewhere that doesn't punish me for trying?*

That's the question this article is built around. We'll walk through what "DigitalOcean Elasticsearch" actually means in practice — because the phrase covers more than one thing — and then get into the deployment paths, the pricing, the security gotchas, and the moment where you should stop doing it yourself and let someone else run the cluster. Along the way I'll point you at the specific plans and credits that matter, so you can make the call with numbers in front of you instead of vibes.

## What "DigitalOcean Elasticsearch" Actually Refers To

Here's the first thing that confuses people: DigitalOcean doesn't sell a product literally called "Managed Elasticsearch." What they offer is a few different shapes that all happen to solve the same underlying problem — getting a Lucene-based search and analytics engine running on infrastructure that bills predictably.

There are essentially three paths, and which one you pick depends on how much of the operational burden you want to keep.

**Path one — self-host on a Droplet.** You spin up a Linux virtual machine, install Elasticsearch from Elastic's APT repository, configure `elasticsearch.yml`, open port 9200 carefully, and you own everything from there. This is the classic "I'll do it myself" route. It's the cheapest on paper and the most expensive in 2 a.m. pager incidents.

**Path two — the Marketplace 1-Click.** DigitalOcean's Marketplace has an "ElasticSearch" 1-Click App on Ubuntu 24.04 that drops a preconfigured instance in about a minute. There's also an "ELK Blueprint" that stands up three Droplets with Elasticsearch, Logstash, and Kibana wired together. You still operate the stack, but the install friction is gone.

**Path three — Managed OpenSearch.** This is the one most teams actually want once they've been burned by self-hosting. OpenSearch is the Apache 2.0 fork of Elasticsearch 7.10.2, created by Amazon in 2021 after Elastic changed its license. The engines share the same Lucene core, the same REST API shape, and most client libraries work against both. DigitalOcean runs OpenSearch as a fully managed database — they handle provisioning, backups, scaling, failover, and patches. You get a connection string and a bill.

> A lot of "DigitalOcean Elasticsearch" searches are really about OpenSearch and people just don't know the name changed. If you're coming from an older Elastic stack, the migration is real but usually not painful — snapshot/restore or reindex-from-remote handles versions up to 7.10.x cleanly. For 8.x and 9.x you'll need to adapt some queries because the APIs have diverged.

So when we talk about "DigitalOcean Elasticsearch" in the rest of this piece, we're really talking about all three of these, with the managed OpenSearch option being the one I'd point most readers toward first. If you want to poke at any of them, 👉 [start with a DigitalOcean account here and grab the new-user credit](https://bit.ly/DigitaLocean) — that credit applies to Droplets, Marketplace apps, and Managed OpenSearch alike.

## Why People End Up Looking at DigitalOcean for Search in the First Place

The honest reason is usually price. Elasticsearch clusters have a way of growing into something you didn't plan for — you start with one node for full-text search on your product catalog, then someone adds log aggregation, then someone else wants vector search for the new RAG feature, and suddenly you're paying for a distributed system that nobody on your team actually enjoys maintaining.

DigitalOcean's pitch is that the math is simpler. Droplets start at $4 a month for a 512 MiB / 1 vCPU box with 500 GiB of transfer, and the pricing is flat across regions — you don't get the "wait, that region costs more?" surprise that shows up on bigger clouds. Managed databases follow the same philosophy: monthly caps, no bandwidth bill for traffic to and from managed databases, and storage that scales in 10 GiB chunks at $0.21 per GiB per month.

There's also a developer-experience angle that's harder to quantify but real. The control panel is small and readable. The docs are written like a person wrote them. The community tutorial library — which is where a lot of the canonical "how to install Elasticsearch on Ubuntu" content lives — is genuinely one of the better free resources on the internet for this stuff. If you're a solo dev or a small team that doesn't have a dedicated platform engineer, that matters more than people admit.

## Self-Hosting Elasticsearch on a Droplet: The Cheap and Honest Path

Let's start with the path that costs the least and teaches you the most. Self-hosting Elasticsearch on a Droplet is what most of the existing "DigitalOcean Elasticsearch" tutorials describe, and it's a perfectly reasonable choice for dev, staging, low-traffic production, or anything where you want full control over the configuration.

### Picking the Right Droplet Size

Elasticsearch is a memory-hungry beast. The official guidance is to give the JVM heap about half the machine's RAM and let the OS use the rest for the filesystem cache — Lucene loves having hot segments in page cache. So a 1 GiB Droplet is going to be a sad experience for anything beyond a toy. Realistically:

- **2 GiB / 1 vCPU / 50 GiB SSD — $12/mo.** The minimum I'd consider for a single-node dev Elasticsearch. Heap gets 1 GiB, OS gets 1 GiB. Fine for testing queries against a few hundred thousand docs.
- **4 GiB / 2 vCPUs / 80 GiB SSD — $24/mo.** A reasonable small-production single node. Two vCPUs matters because Elasticsearch does real work on the indexing path.
- **8 GiB / 4 vCPUs / 160 GiB SSD — $48/mo.** This is where you can start thinking about a small cluster or a single node that handles real traffic.
- **16 GiB / 8 vCPUs / 320 GiB SSD — $96/mo.** A serious single node or a comfortable cluster member.

If you're going to run the full ELK stack (Elasticsearch + Logstash + Kibana) on one box, Kibana and Logstash each want their own chunk of RAM, so bump up a size class from what you'd pick for Elasticsearch alone. The Marketplace ELK Blueprint exists partly because doing this by hand on a small box is a recipe for OOM kills.

For workloads where the data is the point — large indexes, lots of segments, log analytics — the Memory-Optimized and Storage-Optimized Droplet lines exist for exactly this. Memory-Optimized gives you 8 GiB of RAM per vCPU (so a 16 GiB / 2 vCPU box is $84/mo), and Storage-Optimized uses NVMe SSDs that are an order of magnitude faster on disk I/O. If you're indexing hundreds of GB of logs a day, that NVMe difference is the difference between "queries return" and "queries time out."

### The Actual Install Steps (Short Version)

The full walkthrough is in DigitalOcean's community tutorial on installing Elasticsearch on Ubuntu 22.04, but the shape of it is:

1. Import Elastic's GPG key and add their APT source.
2. `apt install elasticsearch`.
3. Edit `/etc/elasticsearch/elasticsearch.yml` — at minimum, set `network.host: localhost` so you don't accidentally expose port 9200 to the open internet (this is the single most common Elasticsearch security disaster, and it happens because the default used to bind to all interfaces).
4. `systemctl start elasticsearch && systemctl enable elasticsearch`.
5. `curl -X GET 'http://localhost:9200'` — if you get the cluster info JSON back, you're up.

From there you can add data with `curl -XPOST`, query it with `curl -X GET`, and start building the rest of your app against it. The REST API is the same one every Elasticsearch client in every language speaks, so your application code doesn't care that it's running on a $12 Droplet.

### The Security Conversation Nobody Wants to Have

Here's the part that bites people. Out of the box, a self-hosted Elasticsearch has no authentication. Anyone who can reach port 9200 can read your data, delete your indexes, and shut down your cluster through the REST API. There's a long, sad history of exposed Elasticsearch clusters leaking data because someone set `network.host: 0.0.0.0` and walked away.

The minimum you should do:

- Bind to `localhost` or a private network interface, never the public IP, unless you've put something in front of it.
- Use UFW to lock port 9200 down to specific trusted IPs: `sudo ufw allow from <your-app-server-ip> to any port 9200`.
- If you need remote access over the public internet, put Nginx in front as a reverse proxy with SSL and HTTP basic auth, or use a VPN / WireGuard mesh between your nodes.
- For multi-node clusters on a shared private network, use a VPN — Elasticsearch's transport protocol isn't encrypted by default in the open-source distribution.

Elastic's commercial Shield / X-Pack security features (authentication, RBAC, audit logging, encrypted internode traffic) are part of the paid tiers now. If you need those without paying, that's one of the reasons OpenSearch exists — it ships all of that free under Apache 2.0. We'll come back to that.

## The Marketplace 1-Click Option: Less Friction, Same Responsibility

If "install APT packages and edit YAML" sounds like a chore, the Marketplace has two relevant 1-Click apps.

**ElasticSearch 1-Click** drops a preconfigured Elasticsearch instance on Ubuntu 24.04 in about a minute. You pick the Droplet size and region, and you get a working single node without touching APT. You still own the security configuration and the operational burden — it's just the install step that's been done for you.

**ELK Blueprint** is the bigger one. It deploys three Droplets with Elasticsearch, Logstash, and Kibana wired together, intended for teams that want the full log-analytics stack out of the gate. Three nodes is the minimum for a real Elasticsearch cluster with quorum and high availability, so this is the first option on the list that gets you past "single node that dies if the box dies."

You can 👉 [deploy the Elasticsearch 1-Click or ELK Blueprint from the Marketplace through this link](https://bit.ly/DigitaLocean) — the new-user credit applies to Marketplace apps the same way it applies to plain Droplets.

The thing to understand about Marketplace apps is that they're convenience, not managed service. You're still the one doing backups, monitoring disk usage, applying security patches, and noticing when the cluster is about to fall over. If that sentence made you tired, the next section is for you.

## Managed OpenSearch: The "Stop Operating It" Path

This is the option I'd recommend most teams at least price out before self-hosting. DigitalOcean's Managed OpenSearch is a fully managed database service — they handle cluster provisioning, automated backups (hourly for the first 24 hours, then daily with up to 3 days retention), software updates, monitoring, and failover. You get a connection string, you point your client at it, you go back to building your product.

### Why OpenSearch Instead of Elasticsearch?

The short version: licensing and cost. In January 2021, Elastic moved Elasticsearch off Apache 2.0 to a dual SSPL / Elastic License 2.0 model. Amazon forked the last Apache 2.0 version (7.10.2) into OpenSearch, which is now governed by the OpenSearch Software Foundation under the Linux Foundation. The two engines share the Lucene core and most of the API surface, but they've diverged in ways that matter:

- **OpenSearch ships enterprise security free.** LDAP, Active Directory, SAML, OpenID Connect, role-based access control, field-level security, document-level security, audit logging — all of it is built in under Apache 2.0. Elasticsearch includes basic security and RBAC in its free tier, but LDAP/SAML SSO, document- and field-level security, and audit logging require paid subscriptions.
- **Cross-cluster replication and searchable snapshots are free in OpenSearch**, paid (Enterprise tier) in Elasticsearch.
- **Vector search differs.** OpenSearch supports both Lucene HNSW and Faiss as vector engines, with a max of 16,000 dimensions. Elasticsearch uses Lucene HNSW only, capped at 4,096 dimensions, but has more mature quantization (BBQ, DiskBBQ). For RAG workloads with high-dimensional embeddings, OpenSearch's Faiss support is a real differentiator.
- **Managed service competition.** Only Elastic sells managed Elasticsearch (via Elastic Cloud). OpenSearch has multiple managed providers — Amazon OpenSearch Service, Aiven, Instaclustr, and DigitalOcean — competing on price. More competition means lower prices, typically 30-50% cheaper than Elastic Cloud on equivalent workloads.

For most teams doing full-text search, log analytics, or app search, the engine-level differences are marginal. The decision usually comes down to cost and which ecosystem your team already lives in. If you're starting fresh and cost matters, OpenSearch is the rational default.

### What You Get With DigitalOcean Managed OpenSearch

The feature list, pulled from the official product page:

- **Fully managed cluster provisioning** — set up in minutes, choose region, size, and node count.
- **Automated backups** — hourly for the first 24 hours, then daily with up to 3 days retention.
- **High availability** — automated failover, monitoring, and the option to add standby nodes.
- **VPC isolation** — clusters run in your private network, with optional IP whitelisting for public internet access.
- **Encryption** — in transit and at rest, included.
- **Autoscaling storage** — add storage or nodes without rebuilding the cluster.
- **Log forwarding** — forward logs from other DigitalOcean resources (Managed Databases, App Platform, Kubernetes) directly into your OpenSearch cluster. There's also Datadog integration for managed database logs.
- **OpenSearch Dashboards** — the Kibana equivalent, for visualization and query building.

The user reviews on the product page are telling. One quote that stuck out: a team at Gleap migrated from self-hosted Elasticsearch to Managed OpenSearch and reported that "it allowed us to keep our OpenSearch-compatible workloads close to the majority of our services, massively reducing latency while also improving security across our platform." Another team running 20k database operations per second said the log forwarding into OpenSearch "is really brilliant and helps us a lot to identify issues or slow DB queries."

That second quote is the use case that makes managed OpenSearch worth it: centralized log analytics across your whole DigitalOcean footprint, without you running the log cluster.

## The Full Plan Comparison: Managed OpenSearch Pricing

Here's the part you came for. These are the shared-CPU Managed OpenSearch plans currently listed on DigitalOcean's pricing page. Storage is billed separately at $0.21 per GiB per month, in 10 GiB increments, on top of the base plan price. Traffic to and from managed databases doesn't count against your bandwidth allowance.

| Plan | RAM | vCPUs | Storage Range | Hourly | Monthly | Get Started |
| --- | --- | --- | --- | --- | --- | --- |
| **OpenSearch 2 GiB (shared)** | 2 GiB | 1 vCPU | 40–200 GiB | $0.02917 | $19.60 | [Deploy this plan](https://bit.ly/DigitaLocean) |
| **OpenSearch 4 GiB (shared)** | 4 GiB | 2 vCPUs | 70–350 GiB | $0.05513 | $37.05 | [Deploy this plan](https://bit.ly/DigitaLocean) |
| **OpenSearch 8 GiB (shared)** | 8 GiB | 4 vCPUs | 150–750 GiB | $0.11347 | $76.25 | [Deploy this plan](https://bit.ly/DigitaLocean) |
| **OpenSearch Dedicated (starting)** | — | Dedicated | — | — | from $97.60 | [Deploy this plan](https://bit.ly/DigitaLocean) |
| **OpenSearch 3-node cluster (shared vCPUs)** | — | Shared | — | — | $111.15 | [Deploy this plan](https://bit.ly/DigitaLocean) |

A few notes on reading this table:

- The $19.60/mo entry point is for a single-node cluster with 2 GiB of RAM. That's the cheapest way to get a managed OpenSearch cluster running. It's fine for dev, small log analytics, or a search backend for a low-traffic app.
- The dedicated plans start at $97.60/mo and give you dedicated CPUs (no noisy neighbors). The exact specs scale from there — the product page lists dedicated options up through larger configurations for production workloads.
- The 3-node shared vCPU cluster at $111.15/mo is the minimum for real high availability. Three nodes means quorum, which means the cluster survives a node failure without losing data. If you're running production, this is the floor.
- Storage is the variable cost. A 2 GiB plan with 200 GiB of storage runs $19.60 + (200 × $0.21) = $61.60/mo. A 4 GiB plan with 350 GiB runs $37.05 + (350 × $0.21) = $110.55/mo. Model your data size before you commit.
- All plans include the managed service layer — backups, monitoring, failover, security. There's no separate "support tier" upcharge for the basics.

If you want to spin one up and see the actual numbers for your workload, 👉 [head to the Managed OpenSearch creation flow through this link](https://bit.ly/DigitaLocean) — the new-user credit applies, so your first cluster is effectively free for the credit window.

## Droplet Pricing for the Self-Hosting Route

For completeness, here's the Droplet pricing you'd be looking at if you go the self-hosted route instead. These are the relevant lines for an Elasticsearch workload — Basic for dev, Memory-Optimized for production single nodes, Storage-Optimized for big log indexes.

| Droplet Type | RAM | vCPUs | SSD | Transfer | Monthly | Best For |
| --- | --- | --- | --- | --- | --- | --- |
| **Basic** | 512 MiB | 1 | 10 GiB | 500 GiB | $4 | Toy / testing |
| **Basic** | 1 GiB | 1 | 25 GiB | 1,000 GiB | $6 | Small dev |
| **Basic** | 2 GiB | 1 | 50 GiB | 2,000 GiB | $12 | Min viable ES dev |
| **Basic** | 4 GiB | 2 | 80 GiB | 4,000 GiB | $24 | Small prod single node |
| **Basic** | 8 GiB | 4 | 160 GiB | 5,000 GiB | $48 | Real single node |
| **Basic** | 16 GiB | 8 | 320 GiB | 6,000 GiB | $96 | Cluster member |
| **Memory-Optimized** | 16 GiB | 2 | 50 GiB NVMe | 4,000 GiB | $84 | Heap-heavy single node |
| **Memory-Optimized** | 32 GiB | 4 | 100 GiB NVMe | 6,000 GiB | $168 | Comfortable prod node |
| **Storage-Optimized** | 16 GiB | 2 | 300 GiB NVMe | 4,000 GiB | $131 | Log analytics, big indexes |
| **Storage-Optimized** | 32 GiB | 4 | 600 GiB NVMe | 6,000 GiB | $262 | Heavy log ingestion |

The pattern is pretty clear: a self-hosted Elasticsearch on a $24 Basic Droplet costs less than the cheapest managed OpenSearch ($19.60 + storage), but you're trading money for hours of your life spent on operations. The crossover point where managed becomes cheaper than self-hosted is usually around the moment you need a second node for high availability — at that point you're paying for two Droplets, doing your own backups, and the $111.15/mo 3-node managed cluster starts looking like a bargain.

Effective January 1, 2026, Droplets moved to per-second billing with a 60-second minimum, which makes short-lived workloads (batch jobs, automated testing, ephemeral dev clusters) dramatically cheaper. For a long-running production Elasticsearch that's not a factor, but for CI and experimentation it's a real change.

## Self-Hosted vs Managed: The Actual Decision

People ask "should I self-host or use managed?" like it's a moral question. It's not — it's a math question with a time component.

**Self-host on a Droplet if:**

- You're running dev / staging / a low-traffic production search backend.
- You want full control over the Elasticsearch version, plugins, and configuration.
- You're comfortable doing your own backups, monitoring, and security hardening.
- Your team has at least one person who's run Elasticsearch in production before.
- Cost is the dominant constraint and your time is cheap.

**Use Managed OpenSearch if:**

- You're running production and uptime matters.
- You're doing log analytics across multiple DigitalOcean resources (the log forwarding feature is a real productivity win here).
- You need enterprise security features (LDAP, SAML, field-level security) without paying Elastic subscription prices.
- You don't have a dedicated platform engineer and don't want to become one.
- You're doing vector search / RAG and want Faiss support or higher vector dimensions than Elasticsearch's 4,096 limit.

The honest middle ground a lot of teams land on: start self-hosted on a Droplet while you're building, migrate to Managed OpenSearch when you have users and the cluster going down would be embarrassing. The migration path from Elasticsearch 7.10.x to OpenSearch is well-trodden (snapshot/restore or reindex-from-remote), and the client libraries are compatible, so you're not locked in either direction.

## The $200 Free Credit Situation

DigitalOcean runs a referral program that gives new accounts $200 in credit valid for 60 days. This is the single biggest lever for trying any of the above without risk — $200 covers:

- About 10 months of the cheapest managed OpenSearch plan ($19.60/mo).
- About 8 months of a 4 GiB / 2 vCPU Droplet ($24/mo) for self-hosting.
- A decent chunk of a 3-node managed cluster for long enough to load-test it.

The credit applies across the platform — Droplets, Marketplace apps, Managed Databases, storage, bandwidth — so you can try all three paths in the same account and see which one feels right before you spend a dollar of your own money. You can 👉 [claim the $200 credit and start a new account through this link](https://bit.ly/DigitaLocean).

One caveat worth knowing: the credit has a 60-day expiration, so don't spin it up the day before a two-week vacation. Plan to actually use the window.

## A Quick Word on Elasticsearch vs OpenSearch in 2026

Since a lot of people searching "DigitalOcean Elasticsearch" are really trying to figure out the Elasticsearch vs OpenSearch question, here's the short version of where the two projects actually stand right now.

Both shipped major versions in 2025 — Elasticsearch 9.0 in April, OpenSearch 3.0 in May. Both upgraded to Lucene 10. Both have serverless options now (Elastic Cloud Serverless, Amazon OpenSearch Serverless). The core text search and log analytics behavior is broadly comparable on equivalent hardware — for most workloads the engine choice is marginal.

Where they diverge:

- **Licensing**: Elasticsearch is triple-licensed (AGPLv3, SSPL, ELv2). OpenSearch is Apache 2.0. For most internal backend use there's zero licensing risk either way; for embedding in a commercial product or offering as a service, OpenSearch is simpler.
- **Vector search**: OpenSearch supports Faiss with up to 16,000 dimensions and more quantization options. Elasticsearch has more mature quantization (BBQ, DiskBBQ) but is capped at 4,096 dimensions and Lucene HNSW only.
- **Security**: OpenSearch includes LDAP/SAML/field-level security/audit logging free. Elasticsearch puts those behind paid tiers.
- **Ecosystem**: Elasticsearch has the more polished integrated stack (APM, SIEM, Enterprise Search, Kibana, Elastic Agent). OpenSearch is more focused on the core engine and has a broader managed-provider market.
- **Cost on managed services**: OpenSearch is typically 30-50% cheaper than Elastic Cloud for equivalent workloads, partly because of the competing providers.

If you're choosing a managed service on DigitalOcean, the OpenSearch decision is made for you — they don't offer managed Elasticsearch. If you're self-hosting on a Droplet, you can install either. For new projects where cost and permissive licensing matter, OpenSearch is the rational default. For teams already deep in the Elastic ecosystem, staying on Elasticsearch is fine.

## Common Questions People Actually Ask

**Can I run Elasticsearch and Kibana on the same Droplet?** Yes, but it's tight. Kibana wants ~1 GiB of RAM on its own. On a 4 GiB Droplet you'd give Elasticsearch 2 GiB heap, leave 1 GiB for the OS filesystem cache, and Kibana gets squeezed. The Marketplace ELK Blueprint exists partly because doing this on a small box by hand is painful. For anything real, use the ELK Blueprint or split them across two Droplets.

**Is DigitalOcean Managed OpenSearch cheaper than AWS OpenSearch Service?** Often yes, especially for smaller clusters, because DigitalOcean's pricing is flat across regions and includes the managed service layer without separate support tier upcharges. For larger deployments the comparison gets more workload-dependent. The bigger win is usually operational simplicity — DigitalOcean's control panel is dramatically smaller than AWS's, which matters if you don't have a dedicated platform team.

**What happens if my self-hosted Elasticsearch runs out of disk?** It stops accepting writes and goes into read-only mode. This is the single most common self-hosting failure mode. You either add a Block Storage Volume (DigitalOcean's expandable block storage, billed at $0.10/GiB/month for standard tier or $0.30/GiB/month for high-performance tier) or you migrate to a bigger Droplet. Managed OpenSearch handles this with autoscaling storage, which is one of the quieter reasons to use it.

**Can I use the official Elasticsearch clients with OpenSearch?** Mostly yes. The OpenSearch project maintains official clients for Python, Java, JavaScript, Go, Ruby, PHP, .NET, and Rust. The older Elasticsearch 7.x clients also work against OpenSearch in most cases. Newer Elasticsearch 8.x/9.x clients have version checks that may reject OpenSearch connections, so for new code use the OpenSearch clients.

**Does the $200 credit work for Marketplace apps?** Yes. The credit applies to anything billed through DigitalOcean, including Marketplace 1-Click apps, Managed Databases, Droplets, storage, and bandwidth. The Marketplace apps themselves don't have a separate license fee — you pay for the underlying compute.

## The Takeaway

"DigitalOcean Elasticsearch" is really three different things sharing a search phrase. The cheapest path is a Droplet with Elasticsearch installed from APT — $12 to $48 a month gets you a real single node, and the install is well-documented. The lowest-friction path is the Marketplace 1-Click or ELK Blueprint. The path that lets you sleep is Managed OpenSearch, starting at $19.60 a month for a single node or $111.15 for a 3-node HA cluster.

If you're not sure which one you are, the right move is to take the $200 credit, spin up one of each, and find out. The credit window is generous enough that you can run a managed cluster and a self-hosted Droplet side by side for a month and see which one you actually want to maintain. 👉 [Start here with the DigitalOcean credit](https://bit.ly/DigitaLocean) and the rest of the platform is a few clicks away.

The thing about search infrastructure is that the "right" answer changes as you grow. The $12 Droplet that's perfect for dev becomes the production single point of failure you regret at 3 a.m. The managed cluster that felt expensive at $111/mo becomes the thing that saved you from a multi-hour outage. The point of having all three options on one platform is that you can move between them without rewriting your application — the REST API is the same, the clients are the same, the data model is the same. You're not picking a destination, you're picking a starting point.
