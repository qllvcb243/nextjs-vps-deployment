# self host nextjs: run your app on a VPS with Docker or PM2, from standalone build to HTTPS, plus real costs vs Vercel

You don't need Vercel to run Next.js. The framework ships a standard Node.js server, and the official self-hosting guide is explicit about it: `next build` plus `next start` supports every feature on your own hardware — server-side rendering, middleware, image optimization, ISR, all of it. If your site doesn't need a server at runtime, a static export deploys anywhere that can serve HTML, CSS and JS. What stands between you and a live site is a short, boring list: a server, something that keeps the process alive, a reverse proxy that terminates HTTPS, and a repeatable way to ship builds. On a small VPS that stack runs from $2.50 to $6 a month.

> **Short version:** for a typical SSR app, Vultr's High Frequency plan (1 vCPU, 1 GB RAM, 32 GB NVMe, 1 TB transfer, $6/mo) is the sensible default, and it's billed hourly — a test server that lives for two days costs a few cents. A static export fits the $2.50/mo plan. [👉 See Vultr's current plans and hourly pricing](https://bit.ly/VulTR)

## The three ways to self-host Next.js

The official docs list exactly three supported routes: a Node.js server, a Docker container, or a static HTML export. Which one fits comes down to a single question — does your app need to run code per request?

**Node.js server.** Run `npm run build`, then `npm run start`. This path supports all Next.js features and works on any host that offers Node.js, including a plain Ubuntu VPS. This is what most people searching for this topic actually need.

**Docker container.** Same feature set, packaged as an image. This is the route to pick if you deploy to Kubernetes or already run containers anywhere. Build it with `docker build -t nextjs-docker .` and run it with `docker run -p 3000:3000 nextjs-docker`.

**Static export.** Set `output: 'export'` and you get static files you can serve from Nginx, Apache, S3 or any static host. The trade: no middleware, no server-side request handling, and image optimization only works if you define a custom loader in `next.config.js` — and even then, images are optimized at runtime, not during the build. Server Components still work, to be fair.

If your app fetches data per request, uses ISR, or relies on middleware, options three is off the table. Build for a server.

## How much server does Next.js actually need?

Serving takes less than you'd think; building takes more. A 1 GB instance serves most small-to-medium SSR apps comfortably. The `next build` step is the memory hog, and running a build on a 512 MB box will hurt on anything beyond a tiny project. The practical pattern: build locally or in CI, ship the standalone output (more on that below), and let the small server do what it's good at — serving.

Here's the current Vultr lineup relevant to this, with entry configs from the Cloud Compute families:

| Plan | Entry config | Price | Billing | Transfer | Get it |
| --- | --- | --- | --- | --- | --- |
| Cloud Compute – Regular Performance | 1 vCPU, 512 MB RAM, 10 GB SSD | $2.50/mo | Hourly | 0.5 TB | [ Deploy](https://bit.ly/VulTR) |
| Cloud Compute – Regular Performance | 1 vCPU, 1 GB RAM, 25 GB SSD | $5/mo | Hourly | 1 TB | [ Deploy](https://bit.ly/VulTR) |
| Cloud Compute – High Performance | 1 vCPU, 1 GB RAM, 25 GB SSD | $6/mo | Hourly | 1 TB | [ Deploy](https://bit.ly/VulTR) |
| Cloud Compute – High Frequency | 1 vCPU, 1 GB RAM, 32 GB NVMe | $6/mo | Hourly | 1 TB | [ Deploy](https://bit.ly/VulTR) |
| Optimized Cloud Compute (dedicated core) | 1 dedicated vCPU, 8 GB RAM, 25 GB SSD | from $28/mo | Hourly | 1 TB | [ Deploy](https://bit.ly/VulTR) |

Each family scales up to larger CPU and RAM tiers from there, and Vultr also sells bare metal, GPU instances and managed databases if you outgrow a single box — for a Next.js app, the lines above are what you'll actually use. Entry configs and prices change occasionally, so treat the table as a snapshot and check the live page before ordering. A few things worth knowing: billing is hourly with no contracts, there are more than 30 data center locations so you can put the server near your users, snapshots are available for point-in-time copies, and automatic backups cost a small add-on fee on top of the instance.

## Server setup, step by step

Everything below assumes a fresh Ubuntu LTS box. [👉 Deploy an Ubuntu server on Vultr](https://bit.ly/VulTR), connect over SSH, then set up the basics:

bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y build-essential curl git ufw
ufw allow OpenSSH
ufw allow http
ufw allow https
ufw enable


Install Node.js LTS with nvm, which keeps version switching painless:

bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
source ~/.bashrc
nvm install --lts


Pull your code onto the server over SSH (add your SSH key to GitHub, then `git clone git@github.com:you/repo.git`) or, better, build off-box as described next.

## Build and run: the standalone output is your friend

Setting `output: "standalone"` changes what `next build` emits:

js
// next.config.js
module.exports = { output: "standalone" };


The build produces `.next/standalone` — a self-contained folder with a minimal `server.js` and only the dependencies actually needed. Copy three things to your server: `.next/standalone`, `.next/static`, and `public`. Then run:

bash
PORT=3000 node server.js


No full `node_modules` on the server, faster starts, and no ambiguity about what got shipped. This is also the correct way to run Next.js inside a Docker container on a VPS — run `node server.js`, not `next start`.

### Keeping it alive with PM2

PM2 restarts the app on crashes, starts it on boot, and gives you logs:

bash
npm install -g pm2
pm2 start npm --name "nextjs-app" -- start
pm2 save
pm2 startup


Run the command PM2 prints after `startup` — that's what registers the boot-time hook. Day-to-day you'll use `pm2 logs`, `pm2 restart nextjs-app`, and `pm2 reload nextjs-app`, which restarts with zero dropped connections. If you'd rather not add a tool, a systemd unit with `Restart=always` does the same job.

### Or ship the whole thing in Docker

If containers are already your workflow, the Dockerfile-plus-VPS route works well on Vultr: build the standalone bundle in the image, expose port 3000, run the container under Docker's restart policy. One bonus from the official docs: because environment variables can be read at runtime on the server during dynamic rendering, a single Docker image can be promoted through staging and production with different values.

## Reverse proxy and HTTPS

A reverse proxy sits in front of Node, terminates SSL, and forwards traffic to `localhost:3000`. Two good options.

**Caddy** is the one-liner, with automatic certificate issuing and renewal:


yourdomain.com {
  reverse_proxy localhost:3000
}


**Nginx + Certbot** if you prefer the classic stack:

nginx
server {
    listen 80;
    server_name your-domain-or-ip;

    location / {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;

        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
}


bash
sudo apt install -y certbot python3-certbot-nginx
sudo certbot --nginx


One ordering detail that trips people up: point your domain's A record at the server IP first and let DNS propagate before requesting the certificate. Certbot will fail otherwise, and the error message won't tell you that's why.

## What Vercel handled that's yours now

Leaving the platform means owning the day-two conveniences. Four of them matter, and each has a fix.

**ISR and caching.** ISR pages and their revalidation share one cache, stored on the server's local filesystem by default — it works automatically when you self-host, on both the Pages and App Router. The catch that most guides skip: run multiple containers or instances and each one gets its own cache, so you'll need to configure a shared or persistent cache location. Also, truly immutable assets (images, hashed bundles) get `Cache-Control: public, max-age=31536000, immutable`, and Next.js won't let you override that — the filenames carry hashes, so it's safe.

**Image optimization.** Works with zero configuration under `next start`. Two caveats from the docs: on glibc-based Linux, sharp may need additional memory-allocator configuration to avoid excessive memory use, and since images are optimized at runtime, the cache eats disk over time. Plan disk accordingly — 25 GB is plenty for a small site, less so for an image-heavy one.

**Middleware.** It runs on the Edge runtime, a subset of Node.js APIs, because it may execute in front of every route. If your middleware needs full Node APIs or a heavy package, move that logic into a layout as a Server Component, or handle it with redirects and rewrites in `next.config.js`. And middleware doesn't work with static exports at all, since it needs the incoming request.

**Preview deploys and rollback.** These become a staging setup you build: your CI pipeline builds, ships the new standalone bundle, restarts behind a health check, and keeps the previous build around for rollback. `pm2 reload` covers zero-downtime restarts; the health-gated deploy script is an afternoon of work, not a project.

## What self-hosting actually costs

Flat and predictable — that's the whole financial argument. A $6/mo Vultr server costs $6 whether it serves 10 requests or 100,000, and hourly billing means short-lived test servers cost cents. Managed deploy platforms sit between raw VPS and Vercel: selfhost.dev, for instance, detects Next.js from GitHub, runs `next build`, and bills around $0.02/hr per project server — roughly $15 a month running 24/7 — with PR previews and one-click rollback included. Vercel itself bills per request, per function, and per GB of egress, so that bill moves with traffic; if your traffic spikes unpredictably, that's the number to model against a flat $6. [👉 Compare Vultr's hourly-billed plans](https://bit.ly/VulTR)

## FAQ

**Can Next.js really run without Vercel?**
Yes. Next.js is open source and ships a standard Node server — `next build` then `next start` (or the standalone server) runs the same app on any machine with Node.js. The managed platform adds edge, ISR and image conveniences on top; useful, not required.

**Does self-hosted Next.js support SSR and middleware?**
Both work with zero configuration under `next start`. Middleware requires a server, so it's off the table for static exports.

**Is 512 MB of RAM enough?**
For serving a static export, yes. For building an SSR app on the box, it's tight — build elsewhere and ship the standalone bundle, or step up to 1 GB.

**Do I need Docker?**
No. Node plus PM2 or a systemd unit is enough for a single app. Docker earns its keep when you already deploy containers or want environment parity across machines.

**How do updates go out without downtime?**
CI copies the new standalone build to the server and `pm2 reload` swaps it in without dropping connections. The next step up is a health-gated deploy script that keeps the old build for instant rollback.

## Picking by situation

Static site or SPA: static export on the $2.50/mo plan, done in an afternoon. Small SSR app on one box: the $6/mo High Frequency plan is the default choice — same price as High Performance with faster NVMe storage. Several apps, heavier traffic, or you want CPU isolation: move up the family tiers or start at the dedicated-core plans from $28/mo. And if you genuinely want GitHub push-to-deploy without writing any deploy config at all, that's managed-deployer territory (the selfhost.dev style of service at ~$0.02/hr) — you pay a premium over a raw VPS for the convenience, which is a legitimate trade if your time is worth more than $9 a month.

Before you order: point your domain's A record early so DNS is ready when Certbot runs, take a snapshot before any OS upgrade, and decide now whether builds happen on the box or off it. Then [👉 spin up a Vultr server and deploy your Next.js app](https://bit.ly/VulTR) — the whole setup above runs on the smallest plan you can resize out of later.
