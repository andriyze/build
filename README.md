# Building & Shipping Web Apps with Claude Code + OpenAI Codex

A practical, opinionated guide to the full loop: **write code with AI agents → push to private GitHub → auto-deploy to a hardened VPS, optionally behind Cloudflare.**

Target audience: developers and DevSecOps engineers who want a low-ops, low-cost, production-grade stack for small-to-medium SaaS projects.

---

## Architecture at a Glance

```mermaid
flowchart TB
    subgraph Local["💻 Local Development"]
        direction LR
        Dev([Developer])
        AI["Claude Code<br/>+<br/>OpenAI Codex CLI"]
        Dev <--> AI
    end

    GH["🔒 Private GitHub Repo<br/>(main branch)"]

    subgraph VPS["🖥️ Ubuntu 24.04 VPS  •  Hetzner / DigitalOcean"]
        direction TB
        TS["🔐 Tailscale<br/>SSH + Coolify UI<br/>(admin only)"]
        CL["⚙️ Coolify<br/>webhook receiver<br/>+ build engine"]
        Docker["🐳 Docker Container<br/>Next.js app"]
        CL -->|build & deploy| Docker
    end

    CF["☁️ Cloudflare <i>(optional)</i><br/>DDoS · WAF · SSL · CDN"]
    User([🌍 End Users])

    subgraph Ext["☁️ Managed Services"]
        direction TB
        DB[("🗄️ PostgreSQL<br/>Supabase / Neon / Managed PG")]
        Auth["🔑 Clerk<br/>Auth & MFA"]
        Email["📧 Resend<br/>Transactional Email"]
        SMS["📱 Telnyx<br/>SMS"]
    end

    Dev -->|"① git push main"| GH
    GH -->|"② webhook on push / merge"| CL
    User -->|"yourdomain.com"| CF
    CF -->|"③ Cloudflare-only 443 (if enabled)"| Docker
    User -.->|"③ direct 443 (if no Cloudflare)"| Docker
    Docker <-->|Supabase client / Prisma optional / SQL| DB
    Docker <--> Auth
    Docker <--> Email
    Docker <--> SMS
    Dev -.->|"SSH over Tailscale"| TS

    classDef cfNode fill:#fed7aa,stroke:#c2410c,stroke-width:2px,color:#0b1220
    class CF cfNode
    style Local fill:#1f2937,stroke:#9ca3af,stroke-width:2px,color:#f9fafb
    style VPS fill:#dbeafe,stroke:#1d4ed8,stroke-width:2px,color:#0b1220
    style Ext fill:#fde68a,stroke:#b45309,stroke-width:2px,color:#0b1220
    linkStyle default stroke:#9ca3af,stroke-width:2px,color:#f9fafb
```

**Read it left-to-right:** you push code from your laptop → GitHub pings Coolify over a webhook → Coolify pulls, builds a Docker image, and swaps the running container. User traffic enters optionally through Cloudflare (and if Cloudflare is enabled, it should be the only thing allowed to hit ports 80/443 on the VPS). The app talks to Postgres via Prisma or using Supabase, plus managed services for auth, email, and SMS. Your own admin access to the box happens over Tailscale — no public SSH.

---

## Table of Contents

- [Philosophy](#philosophy)
- [Part 1 — AI Coding Tools](#part-1--ai-coding-tools)
  - [Claude Code](#claude-code)
  - [OpenAI Codex CLI](#openai-codex-cli)
  - [How to use them together](#how-to-use-them-together)
- [Part 2 — The Tech Stack](#part-2--the-tech-stack)
- [Part 3 — Database Paths: Supabase vs Direct Postgres and Prisma](#part-3--database-paths-supabase-vs-direct-postgres-and-prisma)
- [Part 4 — VPS Setup (Ubuntu 24.04 LTS)](#part-4--vps-setup-ubuntu-2404-lts)
- [Part 5 — Coolify + Docker Auto-Deploy](#part-5--coolify--docker-auto-deploy)
- [Part 6 — Security Hardening](#part-6--security-hardening)
- [Part 7 — Let Claude Code Harden the Box for You](#part-7--let-claude-code-harden-the-box-for-you)
- [Part 8 — End-to-End Workflow](#part-8--end-to-end-workflow)
- [Reference Links](#reference-links)

---

## Philosophy

- **AI writes most of the code.** You direct, review, and own the architecture.
- **Private by default.** Private GitHub repos, private SSH over Tailscale, and optional Cloudflare in front of the public app.
- **Managed where it matters, self-hosted where it's cheap.** Managed auth/email/SMS (you don't want to own deliverability or SOC 2 for SMS), self-hosted app + reverse proxy (you want the $35/mo bill, not the $350/mo one).
- **One-click redeploy.** Every merge to `main` auto-builds and auto-deploys via Coolify webhooks.

---

## Part 1 — AI Coding Tools

Two agents, two ecosystems. Both run in your terminal, both edit files, run commands, and drive git workflows.

### Claude Code

Anthropic's terminal-based coding agent. Built by the same team that trains Claude, so it has the tightest integration with Anthropic's models.

- **Homepage:** https://www.claude.com/product/claude-code
- **Docs:** https://docs.claude.com/en/docs/claude-code/overview
- **npm package:** https://www.npmjs.com/package/@anthropic-ai/claude-code
- **Requires:** Claude Pro, Max, Team, or Enterprise plan, or API credits.

#### Install — macOS & Linux (native installer, recommended)

```bash
curl -fsSL https://claude.ai/install.sh | bash
```

The native installer drops the `claude` binary into `~/.local/bin`, auto-updates in the background, and does **not** require Node.js. This is the method Anthropic officially recommends.

#### Install — macOS via Homebrew

```bash
brew install --cask claude-code
```

Note: Homebrew installs do not auto-update. You'll need `brew upgrade claude-code` periodically.

#### Install — Windows (native installer, PowerShell)

Open **PowerShell** (not CMD) and run:

```powershell
irm https://claude.ai/install.ps1 | iex
```

Native Windows installs require **Git for Windows** — install it first from https://git-scm.com/download/win if you don't have it.

If you prefer Linux tooling on Windows, install WSL2 first (`wsl --install` in an admin PowerShell), then use the macOS/Linux install command inside your Ubuntu WSL shell.

#### Install — npm (all platforms, legacy but supported)

Requires **Node.js 18+**. Use [nvm](https://github.com/nvm-sh/nvm) to install Node — **never use `sudo npm install -g`**.

```bash
npm install -g @anthropic-ai/claude-code
```

#### First run

```bash
cd ~/code/my-project
claude
```

On first launch it opens your browser for OAuth. For CI/CD or headless servers, set `ANTHROPIC_API_KEY` instead.

Useful commands inside the TUI:

```
/help           Show commands
/status         Auth, plan, rate limits
/config         View / edit config
/bug            File a bug
```

And from your shell:

```bash
claude doctor       # Diagnostic report
claude --version
claude mcp list     # List MCP servers
```

---

### OpenAI Codex CLI

OpenAI's terminal coding agent. Runs locally, signs in with your ChatGPT account (Plus, Pro, Business, Edu, Enterprise) or an API key.

- **GitHub:** https://github.com/openai/codex
- **Docs:** https://developers.openai.com/codex/cli
- **Releases (binaries):** https://github.com/openai/codex/releases
- **Platforms:** macOS and Linux fully supported. Windows is supported natively (with sandbox modes) and also runs well in WSL2.

#### Install — macOS & Linux (npm)

```bash
npm install -g @openai/codex
```

#### Install — macOS (Homebrew)

```bash
brew install --cask codex
```

#### Install — Windows

Native Windows is supported, but **WSL2 is the most reliable path** for full sandbox features:

```powershell
# In an admin PowerShell, if you don't have WSL yet
wsl --install
```

Then inside your Ubuntu WSL shell:

```bash
npm install -g @openai/codex
```

For native Windows without WSL, install via npm in PowerShell, or download a prebuilt binary from the [GitHub Releases](https://github.com/openai/codex/releases) page.

#### Install — manual binary (any platform)

Download the archive matching your platform from https://github.com/openai/codex/releases, extract it, rename the executable to `codex`, and move it to somewhere on your `PATH`.

#### First run

```bash
cd ~/code/my-project
codex
```

Select **Sign in with ChatGPT** and complete OAuth in your browser. Or set `OPENAI_API_KEY` to use API billing instead.

---

### How to use them together

The two agents have different strengths. A pragmatic pattern:

| Task | Good fit |
|------|----------|
| Long, multi-file refactors with tight context management | Claude Code |
| Quick "just fix this bug" loops, rapid iteration | Either — pick the one you're faster in |
| Code review of the *other* agent's diff before merge | Whichever you didn't use to write it |
| Plan → review → implement split | Use one for planning, the other for execution |

Practical tip: both tools respect a project-level context file. Claude Code reads `CLAUDE.md`, Codex reads `AGENTS.md`. Keep them in sync (or symlink one to the other) so either agent can pick up where the other left off. A minimal version of this file should include:

- What the project is
- Tech stack and versions
- Directory conventions
- Commands to run tests, lint, typecheck, build
- Any non-obvious gotchas

---

## Part 2 — The Tech Stack

Everything below is chosen for: low monthly cost, minimal ops burden, and good security defaults.

| Layer | Choice | Why |
|-------|--------|-----|
| **Source control** | Private GitHub repo | Webhooks, Actions, dependable |
| **AI coding** | Claude Code + OpenAI Codex CLI | Agentic development in your terminal |
| **Host** | [Hetzner](https://www.hetzner.com/cloud) CX32 / CPX31 (~$7–10/mo, 8 GB) **or** [DigitalOcean](https://www.digitalocean.com/pricing/droplets) Premium 8 GB (~$48/mo) | Hetzner is dramatically cheaper for the same specs; DO if you need US data centers with lower latency |
| **OS** | **Ubuntu 24.04 LTS** | 10-year support window (via Pro), current apt packages, best Docker compatibility |
| **Deploy** | [Coolify](https://coolify.io) | Open-source Heroku/Vercel replacement, git-push-to-deploy, Docker under the hood |
| **Container runtime** | Docker | Standard, everything speaks it |
| **Database path A** | [Supabase](https://supabase.com/pricing) | Fastest path: managed Postgres + auth + storage in one product |
| **Database path B** | Direct Postgres on [Neon](https://neon.tech/pricing), RDS, or another managed Postgres | More control: app-owned schema, cleaner Prisma fit, easier provider swaps |
| **Auth** | [Supabase Auth](https://supabase.com) **or** [Clerk](https://clerk.com/pricing) | Supabase Auth if you want fewer vendors; Clerk if you want best-of-breed auth UX |
| **Transactional email** | [Resend](https://resend.com/pricing) | 3k emails/mo free, clean API, React Email support |
| **SMS / voice notifications** | [Telnyx](https://telnyx.com/pricing) | Cheaper per-SMS than Twilio, solid API |
| **CDN / WAF / DDoS** | [Cloudflare](https://www.cloudflare.com/plans/) (optional) | Free unmetered DDoS mitigation, WAF, cache if you want it |
| **Private network** | [Tailscale](https://tailscale.com/pricing) (free: up to 100 devices) | WireGuard mesh, zero-config SSH, no open public ports |

### Choosing your database path

- **Supabase-native** is the simplest setup. Use Supabase Postgres, Supabase Auth, and Supabase Storage together. Reach for this when you want one dashboard, fewer vendors, and you're happy to use the Supabase client directly.
- **Direct Postgres + Prisma + Clerk** is the cleaner "app owns the data model" setup. Use a plain Postgres provider, let Prisma own schema and migrations, and use Clerk for auth. It's a bit more wiring up front, but you get clearer boundaries and easier portability later.
- **Do not add Prisma to Supabase by default.** Supabase + Prisma works, but it is the more finicky combination. Only add Prisma there if you specifically want Prisma's schema/migration workflow and accept the extra connection-string nuance.
- **Pick one auth owner.** Either Supabase Auth or Clerk should be authoritative for user identity. Avoid running both unless you have a very specific integration reason.

### Recommended app framework

Any modern framework works, but if you don't have a strong preference:

- **Next.js** (App Router) on Node 20+ — plays well with Clerk, Supabase, and Resend out of the box
- **Dockerfile** multi-stage build, `node:20-alpine` base
- **pnpm** for deterministic installs

---

## Part 3 — Database Paths: Supabase vs Direct Postgres and Prisma

There are two sane patterns here:

### Path A — Supabase-native (simplest)

- Use Supabase Postgres, Supabase Auth, Storage, and the Supabase client directly.
- Prisma is **optional**, not required.
- This is the lowest-friction path if you want one vendor and plan to lean on Supabase features like RLS, Auth, or Storage.

### Path B — Direct Postgres + Prisma (more control)

- Use a plain Postgres provider such as Neon, RDS, Crunchy Bridge, or another managed Postgres host.
- Use [Prisma](https://www.prisma.io) for schema, migrations, and typed queries.
- Use Clerk if you want a separate auth provider with a stronger out-of-the-box user-management UX.

**Pragmatic recommendation:** if you want the fewest moving parts, stay Supabase-native and skip Prisma. If you want a cleaner app-owned data layer with better portability and explicit migrations, use direct Postgres + Prisma.

### Why Prisma in the direct-Postgres path

- **End-to-end type safety.** Your schema becomes TypeScript types the moment you run `prisma generate`. No hand-maintained types, no drift between DB and code.
- **Migrations under version control.** `prisma/migrations/` is just SQL files in git. Reviewable, reversible, deployable.
- **Portable across Postgres hosts.** Swap providers without rewriting your app's data access layer.
- **Good escape hatches.** `$queryRaw` lets you drop to SQL for the 5% of queries the query builder can't express cleanly.
- **Prisma Studio** (`pnpm prisma studio`) gives you a browser-based DB explorer during development. Handy for debugging without `psql`.

### Install

```bash
pnpm add -D prisma
pnpm add @prisma/client
pnpm prisma init --datasource-provider postgresql
```

This creates `prisma/schema.prisma` and adds a `DATABASE_URL` line to `.env`.

Docs: https://www.prisma.io/docs/getting-started

### Connection strings: start simple

For a long-lived Node app running on a VPS, start with a normal direct Postgres connection string in `DATABASE_URL`. Prisma already maintains an application-side pool, so you usually do **not** need a provider pooler on day one.

Example `.env`:

```bash
# Recommended starting point for a VPS-hosted Next.js app
DATABASE_URL="postgresql://app:PASSWORD@db.example.com:5432/app?sslmode=require"

# Optional: only if your provider gives you a separate migration URL
DIRECT_URL="postgresql://app:PASSWORD@db.example.com:5432/app?sslmode=require"
```

Use a separate `DIRECT_URL` only when your provider gives you distinct runtime and migration endpoints or when you intentionally introduce a pooler.

If you choose **Supabase + Prisma**, follow Supabase's current Prisma guide exactly instead of cargo-culting generic PgBouncer advice. As of April 2026, Supabase's Prisma docs use the session pooler on `:5432` for server-based Prisma deployments and the transaction pooler on `:6543` for serverless or auto-scaling workloads.

Supabase + Prisma docs: https://supabase.com/docs/guides/database/prisma  
Neon + Prisma docs: https://neon.com/docs/guides/prisma

### `schema.prisma` — minimal working example

```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model User {
  id        String   @id @default(cuid())
  clerkId   String   @unique            // foreign identity from Clerk
  email     String   @unique
  name      String?
  role      Role     @default(USER)
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  scans     Scan[]

  @@index([clerkId])
}

model Scan {
  id        String   @id @default(cuid())
  userId    String
  target    String
  status    ScanStatus @default(PENDING)
  createdAt DateTime @default(now())

  user      User     @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@index([userId, createdAt])
}

enum Role {
  USER
  ADMIN
}

enum ScanStatus {
  PENDING
  RUNNING
  COMPLETED
  FAILED
}
```

If your provider gives you a separate migration URL, extend the datasource block with:

```prisma
directUrl = env("DIRECT_URL")
```

### Migration workflow

During development:

```bash
# Create and apply a new migration from schema changes
pnpm prisma migrate dev --name add_scans

# Regenerate the TypeScript client (usually automatic, but you can force it)
pnpm prisma generate

# Open the browser-based DB explorer
pnpm prisma studio
```

For production, use `migrate deploy` — it applies committed migrations without generating new ones and never prompts.

### Prisma client singleton (Next.js pattern)

Create `lib/prisma.ts` to avoid exhausting connections during hot-reload in dev:

```typescript
import { PrismaClient } from "@prisma/client";

const globalForPrisma = globalThis as unknown as {
  prisma: PrismaClient | undefined;
};

export const prisma =
  globalForPrisma.prisma ??
  new PrismaClient({
    log: process.env.NODE_ENV === "development" ? ["query", "error", "warn"] : ["error"],
  });

if (process.env.NODE_ENV !== "production") globalForPrisma.prisma = prisma;
```

Then anywhere in your app:

```typescript
import { prisma } from "@/lib/prisma";

const user = await prisma.user.findUnique({
  where: { clerkId: auth().userId },
  include: { scans: { orderBy: { createdAt: "desc" }, take: 10 } },
});
```

### Clerk + Prisma: syncing users

If you stay Supabase-native and use Supabase Auth, skip this section and let Supabase own the identity layer.

Clerk owns the identity. Prisma owns the domain data. You bridge them with a webhook.

1. Clerk dashboard → **Webhooks** → add endpoint `https://yourdomain.com/api/webhooks/clerk`.
2. Subscribe to `user.created`, `user.updated`, `user.deleted`.
3. In your Next.js route handler, verify the Svix signature and upsert the `User` row by `clerkId`.

This keeps your Postgres `User` table authoritative for app-specific fields (role, subscription tier, settings) while Clerk handles auth mechanics.

Clerk + Prisma guide: https://clerk.com/docs/users/sync-data

### Running migrations during Coolify deploys

**Recommended:** run migrations as a separate pre-deploy or release step, not inside the app container's `CMD`.

Pre-deployment command:

```bash
pnpm exec prisma migrate deploy
```

That way a bad migration fails the deploy *before* the new container starts taking traffic.

Do **not** hide `prisma migrate deploy` inside your runtime startup command unless the runtime image intentionally includes the Prisma CLI and the `prisma/` directory. The small standalone Next.js image from Part 5 is for serving the app, not for carrying every build-time tool.

### AI tip — make `schema.prisma` part of Claude Code / Codex context

Your Prisma schema is the single best piece of context you can hand a coding agent. Either:

- Keep it referenced from `CLAUDE.md` / `AGENTS.md`, or
- Just `@prisma/schema.prisma` it into the prompt when asking for new queries, migrations, or seed scripts.

Agents are excellent at writing type-safe Prisma queries, generating realistic seed data, and drafting migration SQL — all of which are boring and error-prone to do by hand.

---

## Part 4 — VPS Setup (Ubuntu 24.04 LTS)

Pick a box. Starting point:

- **Hetzner CPX31** — 4 vCPU / 8 GB / 160 GB SSD — around **$10/mo** (EU/US locations available). https://www.hetzner.com/cloud
- **DigitalOcean Premium Intel 8 GB** — around **$48/mo** (US/EU/APAC). https://www.digitalocean.com/pricing/droplets

At provisioning time:

1. **OS:** Ubuntu 24.04 LTS (x86_64).
2. **SSH key:** upload your public key — do **not** use password auth.
3. **Location:** closest to your users, or closest to you if latency for admin matters.
4. **Name + tags:** meaningful, you'll forget.

### First-boot hardening

SSH in as `root`, then:

```bash
# 1. Create a non-root user and copy your SSH key
adduser deploy
usermod -aG sudo deploy
install -d -m 700 -o deploy -g deploy /home/deploy/.ssh
cp /root/.ssh/authorized_keys /home/deploy/.ssh/authorized_keys
chown deploy:deploy /home/deploy/.ssh/authorized_keys
chmod 600 /home/deploy/.ssh/authorized_keys

# 2. Update everything
apt update && apt upgrade -y
apt install -y ufw fail2ban curl git

# 3. Enable fail2ban
systemctl enable --now fail2ban
```

Before touching `sshd_config`, open a **second terminal** and confirm `ssh deploy@your-server` works and `sudo -v` succeeds.

Only then harden SSH:

```bash
cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak
sed -i 's/^#\?PermitRootLogin.*/PermitRootLogin no/' /etc/ssh/sshd_config
sed -i 's/^#\?PasswordAuthentication.*/PasswordAuthentication no/' /etc/ssh/sshd_config
sshd -t
systemctl restart ssh
```

Keep the original root session open until you've verified the `deploy` login still works after the restart.

### Install Docker

Use Docker's official script (Coolify needs Docker):

```bash
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker $USER
newgrp docker
docker --version
```

Reference: https://docs.docker.com/engine/install/ubuntu/

---

## Part 5 — Coolify + Docker Auto-Deploy

[Coolify](https://coolify.io) is an open-source self-hosted PaaS. Think "Vercel/Heroku, but the bill is your VPS cost." It handles:

- Git webhook → build → deploy pipelines
- Docker / Dockerfile / Nixpacks builds
- TLS via Let's Encrypt (or Cloudflare origin certs)
- Env var management
- Rollbacks

### Install Coolify

```bash
curl -fsSL https://cdn.coollabs.io/coolify/install.sh | sudo bash
```

Official docs: https://coolify.io/docs/get-started/installation

After install, open `http://<server-ip>:8000` and create your admin account immediately. (We'll close this port to the public later — it'll be reachable over Tailscale only.)

### Connect GitHub

In Coolify:

1. **Sources → GitHub App → Create new**. This walks you through creating a GitHub App with repo read + webhook permissions.
2. Install the app on your private repo(s).
3. Coolify now receives push/PR webhooks automatically.

Docs: https://coolify.io/docs/knowledge-base/git/github

### Create your first app

1. **New Resource → Public/Private Repository**.
2. Pick your GitHub App + repo + branch (`main`).
3. Build pack: **Dockerfile** (recommended) or **Nixpacks** for zero-config.
4. Set env vars (Clerk keys, Supabase or Postgres connection string, Resend API key, Telnyx creds).
5. Set domain: `app.yourdomain.com`.
6. Enable **Auto Deploy on git push**.
7. **Deploy**.

Every push to `main` now rebuilds the Docker image on the VPS and does a rolling swap. Merge a PR → ~60 seconds later it's live.

### Sample `Dockerfile` (Next.js)

This assumes your `next.config.js` contains `output: "standalone"`.

```dockerfile
FROM node:20-alpine AS deps
WORKDIR /app
COPY package.json pnpm-lock.yaml ./
RUN corepack enable && pnpm install --frozen-lockfile

FROM node:20-alpine AS build
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN corepack enable && pnpm build

FROM node:20-alpine AS runner
WORKDIR /app
ENV NODE_ENV=production
COPY --from=build /app/.next/standalone ./
COPY --from=build /app/.next/static ./.next/static
COPY --from=build /app/public ./public
EXPOSE 3000
CMD ["node", "server.js"]
```

---

## Part 6 — Security Hardening

This is where a $10 box becomes a defensible production host.

### Layer 1 — UFW firewall baseline

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Temporarily allow SSH from anywhere — we'll lock this to Tailscale next
sudo ufw allow 22/tcp

# HTTP/HTTPS — leave these public unless you later choose Cloudflare-only origin lockdown
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp

sudo ufw enable
sudo ufw status verbose
```

### Layer 2 — Tailscale for admin access

Install Tailscale and kill public SSH.

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
sudo tailscale set --ssh
```

Docs: https://tailscale.com/docs/install/linux

Now on any of your laptops/phones, install Tailscale and log in with the same account. Your VPS shows up as `your-vps` in your tailnet with a `100.x.y.z` IP.

Lock SSH and Coolify to the Tailscale interface only:

```bash
# Remove public SSH rule
sudo ufw delete allow 22/tcp

# Allow SSH + Coolify UI only over Tailscale
sudo ufw allow in on tailscale0 to any port 22 proto tcp
sudo ufw allow in on tailscale0 to any port 8000 proto tcp

sudo ufw reload
```

From now on: `ssh deploy@your-vps` works only when Tailscale is connected. No brute force surface on the public internet.

### Layer 3 — Cloudflare in front of 80/443 (optional)

This layer is optional. If you do **not** want Cloudflare, stop after Layer 2 and leave 80/443 publicly reachable.

Point your domain at Cloudflare, then point an A record for `app.yourdomain.com` at your VPS IP with the **orange cloud** (proxy) enabled. All traffic now hits Cloudflare first.

In the Cloudflare dashboard:

- **SSL/TLS mode:** Full (strict). Install a Cloudflare Origin Certificate on your VPS (Coolify supports custom certs), or let Coolify issue a Let's Encrypt cert and keep CF in Full mode.
- **Always Use HTTPS:** on.
- **Rules → WAF:** enable managed rules (free plan includes the basic OWASP set).
- **Security → Bots:** enable bot fight mode.

### Layer 4 — Only allow Cloudflare IPs on 80/443 (only if Cloudflare is enabled)

This is the step most people skip. It blocks direct browser traffic to the origin even if someone discovers your VPS IP.

By itself, IP allowlisting only proves "this request came from the Cloudflare network". If you want the origin to cryptographically verify Cloudflare on HTTPS, also enable **Authenticated Origin Pulls** and prefer a zone-level or per-hostname certificate over the global shared one.

Cloudflare publishes its IP ranges here: https://www.cloudflare.com/ips/

First, remove the wide-open rules:

```bash
sudo ufw delete allow 80/tcp
sudo ufw delete allow 443/tcp
```

Then create `/usr/local/bin/refresh-cloudflare-ufw.sh` so updates are **idempotent** instead of append-only:

```bash
#!/usr/bin/env bash
set -euo pipefail

tmp4="$(mktemp)"
tmp6="$(mktemp)"
trap 'rm -f "$tmp4" "$tmp6"' EXIT

curl -fsSL https://www.cloudflare.com/ips-v4 -o "$tmp4"
curl -fsSL https://www.cloudflare.com/ips-v6 -o "$tmp6"

# Delete previously managed rules from highest number to lowest so indices do not shift.
mapfile -t managed_rules < <(
  sudo ufw status numbered \
    | sed -n 's/^\[[[:space:]]*\([0-9]\+\)\].*cloudflare-origin.*/\1/p' \
    | sort -rn
)

for rule in "${managed_rules[@]}"; do
  sudo ufw --force delete "$rule"
done

while read -r ip; do
  [ -n "$ip" ] || continue
  sudo ufw allow from "$ip" to any port 80 proto tcp comment 'cloudflare-origin'
  sudo ufw allow from "$ip" to any port 443 proto tcp comment 'cloudflare-origin'
done < "$tmp4"

while read -r ip; do
  [ -n "$ip" ] || continue
  sudo ufw allow from "$ip" to any port 80 proto tcp comment 'cloudflare-origin'
  sudo ufw allow from "$ip" to any port 443 proto tcp comment 'cloudflare-origin'
done < "$tmp6"

sudo ufw reload
sudo ufw status numbered
```

Save that file, make it executable, run it once, then schedule it weekly:

```bash
sudo chmod 755 /usr/local/bin/refresh-cloudflare-ufw.sh
sudo /usr/local/bin/refresh-cloudflare-ufw.sh
```

```cron
0 4 * * 0 /usr/local/bin/refresh-cloudflare-ufw.sh
```

### Result — attack surface summary (with Cloudflare enabled)

| Port | Public | Tailscale | Cloudflare |
|------|--------|-----------|------------|
| 22 (SSH) | ❌ closed | ✅ open | n/a |
| 8000 (Coolify UI) | ❌ closed | ✅ open | n/a |
| 80 / 443 (app) | ❌ blocked except Cloudflare IPs | n/a | ✅ open |
| everything else | ❌ closed | ❌ closed | ❌ closed |

A normal public scan no longer reaches SSH or the app directly. Your admin panel is reachable only over Tailscale. Your app origin accepts 80/443 only from Cloudflare IPs, and Authenticated Origin Pulls can tighten HTTPS origin verification further.

### Optional — application-level extras

- **Cloudflare Zero Trust / Access** in front of `/admin` routes for a second auth factor.
- **Resend domain verification** with SPF, DKIM, DMARC — set these before your first send.
- **Clerk bot protection + attack protection** — on by default, worth reviewing.
- **Supabase / Neon network restrictions** — allowlist your VPS egress IP so the DB rejects connections from anywhere else.
- **Cloudflare Authenticated Origin Pulls** — worth enabling once the basics are stable.

---

## Part 7 — Let Claude Code Harden the Box for You

You already have Claude Code installed on your laptop. You can also install it **on the VPS itself** and hand it the hardening checklist from Part 6 — or a much longer one — and let it execute the steps under your supervision.

This is surprisingly effective. It's also one of the places where being a careful operator matters most: you're giving an AI agent `sudo` on a server that will hold production data. Below is a threat-aware way to do it.

### Two models: agent on laptop vs. agent on VPS

| Approach | Pros | Cons |
|----------|------|------|
| **A. Claude Code on your laptop, SSH out to the VPS** | No API key on the server. Session transcript stays on your machine. Easy to `Ctrl+C` the SSH. | Agent has to reason about remote state indirectly (runs `ssh vps "cmd"` for every step). Slower loop. |
| **B. Claude Code installed on the VPS, you SSH in and run it** | Agent reads files and runs commands natively — much tighter iteration. | API key or OAuth session lives on the server. Blast radius of a compromised server is larger. |

**Recommendation:** Use approach **A** for first-time hardening (before the server holds anything valuable), then switch to **B** for day-to-day admin once the box is locked down. Or stick with A indefinitely — there's no rule you have to run the agent on the server.

### Installing Claude Code on the VPS (approach B)

SSH in via Tailscale, then run the same native installer as on your laptop:

```bash
ssh deploy@your-vps     # Tailscale hostname or 100.x.y.z
curl -fsSL https://claude.ai/install.sh | bash

# Make sure ~/.local/bin is on PATH (installer usually adds it)
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
claude --version
```

#### Authentication on a headless box

You have three options:

1. **OAuth with device-code flow.** Run `claude`; it prints a URL. Open it in your laptop browser, sign in, paste the code back. Uses your Pro/Max subscription. Session token is stored in `~/.claude/` on the VPS.
2. **API key via environment variable.** Set `ANTHROPIC_API_KEY` in your shell profile. Pay per token. No interactive step needed — good for cron-style automations, worse as a resident credential.
3. **Don't authenticate on the VPS at all.** Use approach A instead.

If you choose option 2, treat the API key like any other secret: store it in a file with `chmod 600`, never commit it, and rotate it when you're done hardening.

```bash
# Example: API key sourced from a restricted file, not exported globally
echo 'export ANTHROPIC_API_KEY="sk-ant-..."' > ~/.anthropic.env
chmod 600 ~/.anthropic.env
# Then before a session:
source ~/.anthropic.env && claude
```

### The hardening prompt

Create `HARDENING.md` in your home directory on the VPS (or in a repo you've cloned). Claude Code picks up `CLAUDE.md`-style context automatically — having a written spec gives the agent something concrete to execute against, and gives you something to review line-by-line before anything runs.

Example `HARDENING.md`:

```markdown
# VPS Hardening Checklist — Ubuntu 24.04

You are hardening a fresh Ubuntu 24.04 LTS VPS. Work through this checklist
one step at a time. Before running any command, explain what it does and
why. Do NOT batch multiple destructive commands without confirmation.

## Constraints
- Do not modify /etc/ssh/sshd_config and restart sshd in the same step.
  Always ask me to confirm I have a second SSH session open first.
- Do not disable ufw, fail2ban, or Tailscale under any circumstances.
- Do not create new user accounts with passwords; keys only.
- For every sudo command, print it and wait for my approval.
- Keep a running log at /home/deploy/hardening-log.md with timestamp,
  command run, and outcome.

## Tasks
1. Verify Docker is installed and the `deploy` user is in the `docker` group.
2. Install and configure unattended-upgrades for security patches.
3. Configure fail2ban with SSH jail, 5 retries, 1-hour ban.
4. Audit current ufw rules and print a summary table.
5. Set up a weekly cron that rebuilds the Cloudflare-tagged UFW rules
   from Part 6 of the stack guide.
6. Install and configure auditd with a sensible baseline ruleset
   for a web host.
7. Check that Tailscale is running and `tailscale ssh` is enabled.
8. Verify no services are listening on public interfaces other than
   what Cloudflare should reach on 80/443.
9. Produce a final report summarising what changed, what's pending,
   and any anomalies you noticed.
```

### Running it safely

Start in plan mode so nothing executes until you approve:

```bash
cd ~
claude
```

Then inside the TUI:

```
> Please read HARDENING.md and produce a plan. Do not run any commands yet.
```

Claude Code will read the file, produce a step-by-step plan, and show you each command it intends to run. Review it. Then:

```
> Execute step 1 only. Stop and wait for my review before moving to step 2.
```

Work one step at a time. For any command that touches SSH, firewall, or auth, **open a second terminal as a safety net** before approving — so if the agent locks you out you still have a working session to fix it.

### Guardrails that matter

These aren't paranoia, they're the difference between a story about how nice AI-driven admin is and a story about how you had to rebuild the box from a snapshot:

- **Always have a second SSH session open** when modifying `sshd_config`, `ufw`, or firewall rules. Non-negotiable.
- **Snapshot the VPS before the session.** Hetzner and DigitalOcean both offer one-click snapshots. 30 seconds of prep saves hours of recovery.
- **Never approve a "just run everything and report back" prompt.** The whole point of an agent with `sudo` is supervising each step.
- **Log the session.** Claude Code keeps transcripts in `~/.claude/` — keep them. They're your audit trail for what changed.
- **Don't let the agent write SSH keys or create accounts.** Do user management yourself, out of band.
- **After the session, rotate the API key if you used one** and remove it from `~/.anthropic.env`.
- **Diff the result.** Run something like `sudo find /etc -newer /tmp/session-start -type f` before and after, or compare against a known-good etckeeper snapshot, so you can see exactly what was touched.

### What Claude Code is genuinely good at here

- **Reading and explaining** existing config files (`sshd_config`, `ufw status numbered`, `systemctl list-units`) and flagging things that look off.
- **Writing config from a spec** — e.g. "produce an auditd ruleset for a container host" — and explaining each rule.
- **Patching small mistakes quickly** — typos in cron, wrong paths in systemd units, missed `chmod` steps.
- **Generating the boring scripts** — like the Cloudflare-IP refresh script and its cron entry from Part 6.
- **Writing the report** — at the end of the session, "summarise everything you changed, and produce a runbook for the next person" is a genuinely useful final prompt.

### What to keep doing by hand

- Managing users, SSH keys, and sudoers.
- Anything involving secrets: API keys, DB credentials, certificate private keys.
- Irreversible destructive operations (`rm -rf`, `dd`, `mkfs`, partition changes).
- Anything that, if done wrong, locks you out of the box.

### OpenAI Codex CLI as an alternative

Everything above applies almost identically to Codex — install via `npm i -g @openai/codex` on the VPS, authenticate with ChatGPT OAuth or `OPENAI_API_KEY`, and point it at `AGENTS.md` instead of `HARDENING.md`/`CLAUDE.md`. If you use Codex on a production host, keep sandboxing enabled and keep the same "one step at a time" approval discipline.

---

## Part 8 — End-to-End Workflow

What a day looks like once this is wired up:

1. `cd ~/code/my-app` on your laptop.
2. `claude` (or `codex`) — describe the feature, let the agent plan and implement.
3. Review diff, run tests locally, commit.
4. `git push origin feature/foo` → open PR on GitHub.
5. Optional: run the *other* agent as a second reviewer on the diff.
6. Merge to `main`.
7. GitHub fires webhook → Coolify receives it → Docker build starts on the VPS.
8. Coolify swaps containers with zero downtime.
9. If you enabled Cloudflare, it caches static assets automatically.

Roll back: in Coolify, click a previous deployment → **Rollback**. ~10 seconds.

---

## Reference Links

### AI coding tools
- Claude Code docs — https://docs.claude.com/en/docs/claude-code/overview
- Claude Code setup — https://docs.claude.com/en/docs/claude-code/setup
- OpenAI Codex CLI — https://developers.openai.com/codex/cli
- OpenAI Codex GitHub — https://github.com/openai/codex
- OpenAI Codex Windows guide — https://developers.openai.com/codex/windows

### Infra & hosting
- Hetzner Cloud — https://www.hetzner.com/cloud
- DigitalOcean Droplets — https://www.digitalocean.com/pricing/droplets
- Ubuntu 24.04 release notes — https://discourse.ubuntu.com/t/noble-numbat-release-notes/39890
- Docker install on Ubuntu — https://docs.docker.com/engine/install/ubuntu/
- Coolify — https://coolify.io
- Coolify docs — https://coolify.io/docs

### App services
- Supabase — https://supabase.com
- Neon — https://neon.tech
- Clerk — https://clerk.com
- Resend — https://resend.com
- Telnyx — https://telnyx.com

### Security
- Tailscale — https://tailscale.com
- Tailscale install on Linux — https://tailscale.com/docs/install/linux
- Tailscale SSH — https://tailscale.com/kb/1193/tailscale-ssh
- Cloudflare — https://www.cloudflare.com
- Cloudflare IP ranges — https://www.cloudflare.com/ips/
- Cloudflare Authenticated Origin Pulls — https://developers.cloudflare.com/ssl/origin-configuration/authenticated-origin-pull/
- UFW docs — https://help.ubuntu.com/community/UFW
- fail2ban — https://github.com/fail2ban/fail2ban

---

*Ship small, ship often, keep the attack surface small enough to audit in an afternoon.*
