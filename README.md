# n8n vps: Self-Host Workflow Automation Without Per-Execution Fees, Memory Sizing and Real Cost Breakdown

Looking for a VPS to self-host n8n? The short answer: you'll need at least 2 GB RAM and 2 vCPU cores to get started, but **the payload size your workflows handle matters far more than workflow count**. A single workflow processing 50,000 spreadsheet rows will eat more memory than fifty lightweight webhook-to-CRM automations combined.

This isn't about buying the cheapest VPS and hoping for the best. n8n hosting has a few gotchas most providers skip: encryption key backup (lose it and every saved credential goes invalid), execution history eating disk space faster than you'd expect, and webhook stability requiring a reverse proxy you actually need to configure. Plus, there's the question of whether you need queue mode from day one or can start with a single-process setup and scale later.

> **Quick take**: If you're moving off n8n Cloud's $50/month plan, a 2 vCPU / 8 GB RAM VPS costs $8–15/month depending on promotional terms and gives you unlimited executions. That's the entire cost—no per-task fees, no surprise overages. 👉 [Compare VPS options for n8n hosting](https://vps.hosting/?affid=1666)

## Why Self-Host n8n on a VPS Instead of Cloud Plans

Self-hosting makes sense when execution volume climbs. Cloud automation platforms charge per task or per execution—fine for a dozen workflows running hourly, painful when you're syncing CRM data every five minutes or processing webhook batches from payment systems.

With a VPS, the only recurring cost is server rent. Run a million executions a month or ten; the price stays the same. You're paying for compute time and storage, not business activity.

Data sovereignty is the other reason teams pick self-hosting. n8n workflows see everything: customer records, support tickets, invoices, API keys from payment processors. On a VPS you control, that data doesn't transit through third-party infrastructure in jurisdictions you didn't choose.

And you get actual root access. Need a custom Python node? A specific database driver? An internal API behind your firewall? All doable when you own the box.

## VPS Requirements: Memory Trumps CPU for n8n Workloads

n8n spends most execution time waiting—for API responses, database queries, webhook callbacks. It's I/O-bound, not compute-bound. A modest 2-core CPU handles dozens of concurrent workflows without breaking a sweat, because those workflows are mostly idle, waiting for Stripe to respond or Google Sheets to save a row.

**Memory is where things get interesting.** During execution, n8n loads the entire data payload into RAM. A Code node that transforms 100 MB of JSON will temporarily use around 300 MB of memory (it copies the data in, processes it, then copies it out). Multiple heavy workflows running at once can spike memory usage fast.

Here's the sizing breakdown that actually works in production:

| Use Case | RAM | vCPU | Storage | What It Handles |
| --- | --- | --- | --- | --- |
| Personal automations, SQLite, light payloads | 2 GB | 1-2 | 20-40 GB | A few dozen workflows, cron-triggered, minimal data volume |
| Small team, PostgreSQL, modest data loads | 4-8 GB | 2 | 40-100 GB | Up to 50 concurrent executions, webhook-triggered workflows with moderate JSON payloads |
| Queue mode: n8n + workers + Redis | 8 GB | 2-4 | 60-100 GB | Long-running workflows that shouldn't block each other, multiple worker processes |
| Large payloads, file processing, heavy concurrency | 16 GB | 4 | 100-200 GB | Processing multi-MB files, complex data transformations, high-frequency webhook volume |

If the instance is doing anything for a business, **start at 4 GB minimum**. The 2 GB configuration runs n8n fine but leaves almost no room when PostgreSQL and a payload spike happen at the same time. Providers like Hostinger offer 8 GB RAM / 2 vCPU configurations at $8.79/month (promotional 24-month term, renews at $14.99/month), which gives you comfortable headroom for team use without overpaying.

## The Four Parts of a Self-Hosted n8n Stack

Every functional n8n VPS setup includes four pieces:

1. **The n8n application itself** – a Node.js process, small when idle, memory-hungry during execution when it's holding workflow data in RAM.

2. **A database** – PostgreSQL for anything you care about. SQLite is the default and works for personal tinkering, but it becomes painful once multiple workflows try to write execution history at the same time. PostgreSQL handles concurrent writes cleanly and doesn't corrupt under load.

3. **Redis + worker processes (queue mode only)** – Not needed for small instances. Essential once you have long-running workflows that would otherwise block the main process. Queue mode splits execution across multiple containers with Redis brokering the work.

4. **A reverse proxy with HTTPS** – Webhooks need a public HTTPS endpoint, and you absolutely cannot expose the n8n editor UI without authentication. Nginx or Caddy handles SSL certificates via ACME and keeps the editor access secure.

## VPS Pricing Landscape: What Different Configurations Actually Cost

Prices vary by provider, promotional terms, and commitment length. Here's what the market looks like:

**Entry-level (2 GB RAM / 1 vCPU / 40-50 GB NVMe)**:
- Promotional rates: $6–9/month on 12-24 month terms
- Standard renewal: $12–15/month
- Suitable for: personal use, fewer than 10 workflows, SQLite database

**Small team tier (4-8 GB RAM / 2 vCPU / 100 GB NVMe)**:
- Promotional rates: $8–18/month on long-term commit
- Standard renewal: $15–30/month
- Suitable for: PostgreSQL, 20-50 workflows, webhook-triggered automations with moderate payload sizes

**Production with queue mode (8-16 GB RAM / 4 vCPU / 200+ GB NVMe)**:
- Promotional rates: $13–35/month
- Standard renewal: $29–60/month
- Suitable for: Redis + workers, concurrent long-running workflows, file processing

Managed VPS plans (where the provider handles OS updates, security patches, and basic troubleshooting) cost 2-4× more than self-managed equivalents. For n8n, you're installing via Docker anyway—managed services don't usually include application-layer support for custom Docker stacks, so you're paying for server management you may not need.

**The renewal price jump is real**: many providers offer 40-70% promotional discounts on first-term billing (12-24 months), then renew at standard rates. A plan advertised at $9/month might renew at $25/month. Check renewal pricing before committing, not just the promotional banner.

👉 [Check current VPS configurations and promotional terms](https://vps.hosting/?affid=1666)

## One-Click Deploy vs. Manual Docker Setup

Some VPS providers offer one-click n8n installation. During VPS provisioning, you select "n8n" from an application list instead of choosing a plain Linux image. The VPS spins up with n8n, PostgreSQL, and a reverse proxy pre-configured. You get a URL, log in, and start building workflows.

This saves 2-4 hours of Docker Compose configuration and SSL certificate setup. The tradeoff: you're locked into the provider's specific n8n stack configuration. Updates, custom environment variables, and queue mode setup may require you to break out of the one-click environment and manage Docker manually anyway.

For most users, the manual Docker Compose route gives you more control. You define the exact n8n version, database settings, and networking in a single `docker-compose.yml` file. Migrating to another VPS later is straightforward—copy your compose file, environment variables, and database dump, then start the stack on the new box.

But if you're testing self-hosting for the first time and want to skip the learning curve, one-click deployment gets you running immediately. You can always rebuild manually once you understand what's actually running.

## The Four Mistakes That Break Self-Hosted n8n Instances

### 1. Losing the Encryption Key

n8n encrypts stored credentials (OAuth tokens, API keys, database passwords) using a generated encryption key. **If you lose this key, every saved credential becomes permanently unrecoverable.** You'll have to manually re-authorize every connected service—Stripe, Google Workspace, Slack, all of it.

The key lives in your `.env` file or n8n config. Back it up separately from the VPS itself. Store it in a password manager, not just on the machine where n8n runs. If the VPS dies and your backup doesn't include the key, you're rebuilding all integrations from scratch.

Most self-hosters discover this the hard way: they restore from a database dump, n8n starts fine, workflows are intact—but every credential shows as invalid because the encryption key didn't make it into the backup.

### 2. Execution History Filling the Disk

By default, n8n saves every successful execution and its full payload. A workflow that runs every 5 minutes with a 1 MB payload writes about 8 GB per month to the database. Three months in, you're out of disk space and PostgreSQL crashes mid-write.

Set retention limits on day one. Configure execution history cleanup in n8n settings: keep successful executions for 7-14 days, errors for 30 days, then auto-delete. Monitor database size weekly. This isn't optional maintenance; it's required unless you want your instance to stop mid-workflow when `/var/lib/docker/volumes` hits 100%.

Some providers charge for additional storage ($4-8/month per 50 GB). If you're processing large files or keeping execution logs for compliance, budget for storage scaling.

### 3. Exposing n8n Without Proper Webhook Setup

Webhooks need a stable public HTTPS endpoint. Each VPS comes with an IPv4 address, but you need a reverse proxy (Nginx or Caddy) with an ACME certificate to provide HTTPS. And you absolutely cannot expose the n8n editor on HTTP without authentication—bots will find it in hours and either try default credentials or probe for known vulnerabilities.

Your domain or subdomain must point to the VPS IP. The reverse proxy handles SSL termination and forwards requests to the n8n container. This isn't automatic; Docker doesn't configure DNS or HTTPS for you.

The consequence of skipping this: webhook-triggered workflows fail intermittently (some services refuse to send webhooks to non-HTTPS endpoints), and your editor UI is either unreachable or exposed to anyone with the IP address.

### 4. Running Workflows Without Execution Timeouts

Without a configured timeout, a workflow waiting on an unresponsive external API will hold memory indefinitely until you manually restart the process. Two of these stuck workflows on a 2 GB instance and the whole thing crashes.

Set reasonable execution timeouts in n8n settings. If a workflow legitimately needs 10 minutes to process a large file, set the timeout to 15 minutes. But don't leave it unlimited and assume every API you call will always respond.

When a workflow hits the timeout, n8n kills it and frees the memory. You'll see the execution marked as failed, but the instance stays stable. Without timeouts, you're one flaky API away from the Linux kernel's OOM killer terminating your entire n8n process.

## Location Choice: Latency and Data Residency

If your n8n workflows connect to systems in a specific region, put the VPS in the same region. An automation server in Singapore talking to APIs in Sydney will have noticeably lower latency than one in Frankfurt making the same calls. The difference shows up when workflows call multiple APIs in sequence—each 200ms of extra latency adds up across 10 API calls.

Data residency is the other consideration. Automation platforms see all the data flowing through them: customer info, payment metadata, support tickets, internal documents. If your business operates under GDPR or has data localization requirements, check where the VPS provider's data centers are.

Many providers operate multiple regions. European locations (Amsterdam, Frankfurt, Bucharest) keep data within EU jurisdiction. US-based data centers (Ashburn, Los Angeles, Dallas) fall under US law. Asia-Pacific locations (Singapore, Tokyo, Sydney) have their own regulatory environments.

Some VPS providers let you choose location during checkout without a price difference. Others charge 10-50% more for premium locations (typically Western Europe and US East Coast) compared to budget regions (Eastern Europe, Asia).

## PostgreSQL, Docker, and Why You Skip SQLite for Anything Real

SQLite is the default database because it requires zero setup. For personal workflows that run once a day and never overlap, it's fine. The moment you have multiple workflows executing concurrently and writing logs to the database at the same time, SQLite starts choking. It doesn't handle concurrent writes well and can corrupt under load.

PostgreSQL handles concurrent writes cleanly. It's more stable under sustained execution volume. And it's required if you ever enable queue mode (multiple worker processes can't share a SQLite file safely).

Docker makes this easy: you run n8n and PostgreSQL as separate containers, defined in a single `docker-compose.yml`. Updates are clean (pull a new image, restart the container). Backups are straightforward (dump the PostgreSQL database nightly and send it off the VPS). And you can reproduce the exact environment on another machine if you need to migrate.

Version pinning matters here. Use explicit Docker image tags (`n8nio/n8n:1.x.x`) instead of `latest` for production. The `latest` tag will auto-update and occasionally introduce breaking changes without warning. Pin your versions, test updates in a staging environment, then roll them to production deliberately.

A working `docker-compose.yml` defines three services: n8n, PostgreSQL, and (optionally) Redis. Mount volumes for persistent data. Set environment variables for database credentials and the encryption key. Expose only the reverse proxy port (80/443) to the internet; keep n8n and PostgreSQL on an internal Docker network.

## Queue Mode: When One n8n Process Isn't Enough

A standard n8n deployment runs one main process. Workflows execute sequentially (or with limited concurrency). If a workflow takes 10 minutes to finish, other workflows wait.

Queue mode splits execution across multiple worker processes, with Redis as the message broker. The main n8n instance receives triggers and queues work. Workers pick up jobs from Redis and execute them independently. Long-running workflows no longer block everything else.

You need this when:
- Workflows regularly take several minutes each
- Webhook volume spikes unpredictably (e-commerce order processing, payment notifications)
- File processing (PDFs, images, large CSVs) is common
- You're running more than 50 concurrent executions regularly

Queue mode adds complexity (you're managing Redis and multiple containers) and uses more memory (each worker holds its own payload during execution). Start with a standard deployment and switch to queue mode when single-process execution starts causing delays.

The memory requirement shifts: instead of one 8 GB instance, you might run one 4 GB main process plus two 2-4 GB workers. Total memory usage is similar, but distributed across processes that can scale independently.

## Real-World Cost Comparison: Cloud vs. Self-Hosted

Let's use numbers from actual deployments:

**n8n Cloud (Starter tier)**: $20-50/month depending on execution volume. Includes 2,500-5,000 workflow executions. Overage charges apply beyond that. A team running 500 executions/day (15,000/month) can easily hit $50-80/month.

**Self-hosted on 8 GB VPS**: $9-25/month depending on promotional terms and renewal status. Unlimited executions. You pay the same whether you run 100 or 100,000 workflows per month.

Break-even happens around 3,000-5,000 executions per month if you count only hard costs. Factor in setup time (4-8 hours for initial Docker configuration, SSL setup, and backup testing), and self-hosting pays off after month 2-3 for most teams.

The hidden cost of self-hosting: you're responsible for updates, security patches, and incident response. If the VPS goes down at 2am, there's no support team to page. If a Docker update breaks your stack, you fix it. For single-person operations or small teams without a dedicated ops person, managed n8n Cloud removes that operational burden.

But for teams that already run their own infrastructure, or where execution volume makes cloud pricing painful, self-hosting drops the recurring cost by 60-80% after the first year.

## Choosing Configuration by Actual Workflow Behavior

Forget the marketing categories ("starter," "professional," "enterprise"). Size your VPS based on what your workflows actually do:

**If workflows mostly hit APIs and wait for responses** (CRM syncs, webhook-to-Slack, scheduled email reports): 4 GB RAM, 2 vCPU. These workflows are idle 95% of the time. Memory usage stays low because payloads are small (JSON objects under 1 MB).

**If workflows transform or process data in Code nodes** (CSV parsing, JSON manipulation, array mapping over thousands of items): 8 GB RAM minimum. Code nodes copy data into memory, transform it, and copy it back out. A 50 MB spreadsheet temporarily uses 150+ MB of RAM during processing.

**If workflows handle files** (PDF generation, image processing, video thumbnail extraction): 16 GB RAM and 100+ GB storage. File operations are memory-hungry, and the files themselves need disk space. A workflow that processes 100 PDFs per day needs room for temporary file storage plus execution logs.

**If workflows run constantly with high concurrency** (processing webhook queues, real-time data pipelines): queue mode with 8 GB main + 4 GB per worker. This distributes load across processes so a slow workflow doesn't block dozens of others.

Don't size based on workflow count. A hundred simple workflows use less memory than five complex data transformation workflows.

## VPS Setup Checklist: What You Actually Need Running

Before you point a domain at your VPS and call it done, verify:

- [ ] Docker and Docker Compose installed
- [ ] `docker-compose.yml` defines n8n + PostgreSQL (+ Redis if queue mode)
- [ ] `.env` file contains database credentials and encryption key
- [ ] Encryption key backed up off the VPS (password manager or separate secure storage)
- [ ] Reverse proxy (Nginx/Caddy) configured with ACME for HTTPS
- [ ] Domain or subdomain DNS points to VPS IP
- [ ] Firewall allows 80 and 443 inbound, blocks direct access to n8n port
- [ ] Execution timeout configured in n8n settings (start with 5-10 minutes)
- [ ] Execution history retention limits set (7-14 days for successful runs)
- [ ] PostgreSQL `pg_dump` running nightly and copying backups off-server
- [ ] Disk usage monitoring active (execution logs grow fast)

Test restore from backup at least once. Spin up a temporary VPS, load your latest `pg_dump`, verify workflows and credentials work. A backup you've never tested is a backup that doesn't exist.

The most common oversight: encryption key not included in the backup process. Database dumps don't contain the key by default—it lives in your `.env` file or n8n config directory. If you back up only the database, restoring will give you workflows that can't decrypt any saved credentials.

## When to Scale Up (and How to Know Before It's Too Late)

Monitor memory during execution, not at idle. An n8n instance sitting idle uses 200-400 MB. That tells you nothing about what happens when three workflows fire at once and each is transforming 50 MB of JSON.

Run your heaviest workflow with real data volumes and watch RAM usage peak. Use `htop` on the VPS or check Docker container stats. If you're consistently hitting 80-90% memory utilization during normal operation, you're one payload spike away from the process getting killed by the kernel's OOM handler.

CPU is rarely the bottleneck. If `htop` shows sustained high CPU usage (above 70% for hours), the problem is usually inefficient workflow logic—nested loops over large arrays, unindexed database queries, or Code nodes doing work that should be pushed to the external API. More vCPUs won't fix algorithmic inefficiency.

Storage fills from execution history. Check database size weekly with `docker exec [postgres-container] du -sh /var/lib/postgresql/data`. If it's growing faster than expected, tighten retention settings or archive old execution data to S3-compatible storage.

Network bandwidth matters only if you're processing large files or hitting APIs with multi-MB responses frequently. A 1 Gbps port (standard on most VPS plans) handles almost all workflow traffic without issue. You'd need to be transferring hundreds of GB per day to saturate it.

## Managed vs. Self-Managed: What You're Actually Paying For

Managed VPS plans include OS updates, security patches, and basic troubleshooting. They cost 2-4× more than self-managed VPS. GoDaddy's managed VPS with 4 GB RAM runs $17.99/month (promotional) renewing at $29.99/month. A self-managed equivalent costs $8-12/month.

The question: what does "managed" cover for n8n hosting?

Usually: server-level management (OS patches, kernel updates, security monitoring). The hosting provider keeps Linux updated and running.

Usually not: application-level support. You're still responsible for n8n updates, Docker configuration, workflow debugging, and database maintenance. If a workflow breaks because of an n8n version incompatibility, managed support won't fix it.

For teams already comfortable with Docker and command-line Linux, self-managed VPS makes sense. You save $10-20/month per server and retain full control. For non-technical teams or solo founders who don't want to learn server administration, managed VPS removes one layer of complexity—though you're still managing n8n itself.

A middle ground: some VPS providers offer "managed applications" where they handle both the server and the application stack. These are rarer and more expensive ($50-100+/month), but they cover n8n updates and troubleshooting. At that price point, n8n Cloud starts looking competitive again.

## Making the Switch from n8n Cloud to Self-Hosted

If you're currently on n8n Cloud and considering a move:

1. **Export all workflows** from the cloud instance (n8n lets you export as JSON).
2. **Document all credentials and OAuth connections** (you'll need to re-authorize them after migration—the encryption key won't transfer).
3. **Spin up a VPS** (start with 8 GB if you're on n8n Cloud's paid tier).
4. **Install n8n via Docker Compose** with PostgreSQL.
5. **Import workflows** and test them one by one with dummy data.
6. **Reconnect all credentials** (OAuth apps, API keys, database connections).
7. **Run parallel for 1-2 weeks**: keep cloud instance active, run new workflows on self-hosted to verify stability.
8. **Update webhooks** to point to your self-hosted domain.
9. **Cancel cloud subscription** once you're confident everything works.

Budget 1-2 days for the migration if you have 20-50 workflows. Most of the time goes to re-authorizing OAuth apps (Google, Microsoft, Salesforce) which require manual login flows.

The biggest surprise for first-time self-hosters: webhook URLs change. If external services send webhooks to your n8n instance (Stripe payment confirmations, form submissions, API callbacks), you need to update those webhook URLs in each external service. Miss one and that integration silently breaks.

👉 [Get started with a self-hosted n8n VPS](https://vps.hosting/?affid=1666)

## Final Sizing Decision: Start Here, Scale When You See Usage

For personal use (under 10 workflows, no business dependencies): 2 GB RAM, 1-2 vCPU, ~$7/month. Use SQLite, skip queue mode, monitor disk space.

For small team use (10-50 workflows, moderate webhook traffic): 8 GB RAM, 2 vCPU, $9-15/month promotional pricing. Use PostgreSQL, start without queue mode, watch memory during peak periods.

For production use with high volume (50+ workflows, file processing, concurrent execution): 16 GB RAM, 4 vCPU, $25-35/month. Use PostgreSQL + Redis + queue mode from day one.

Whatever you pick, actually monitor memory usage during real workflow execution—not idle stats, not marketing benchmarks. Spin up the VPS, load your workflows, run them with production data volumes for a week. If you're comfortable under 70% memory at peak, you're sized right. If you're hitting 90%+ regularly, move up before the instance crashes during a busy period.
