# Interview Prep Notes: Cloud Migration & DevOps Project

## The one-line summary
Led a live, end-to-end migration of a multi-service financial platform (7 Java microservices, 21GB PostgreSQL database, 2 React frontends) from a physical on-premise server to Microsoft Azure, while diagnosing and fixing real production issues along the way — database connection exhaustion, JDK version mismatches, broken CI/CD deployments, and a shared-library dependency conflict.

---

## What to lead with (pick based on the role you're interviewing for)

- **Infrastructure/Cloud role** → lead with the Azure provisioning + database migration story
- **DevOps/SRE role** → lead with the connection-pool incident and the JDK/dependency debugging
- **Platform/Backend role** → lead with the CommonLibrary versioning investigation
- **Full-stack role** → lead with the Static Web Apps + CORS + SPA routing saga

---

## Story 1: The database migration (good "walk me through a project" answer)

**Situation:** Needed to move a 21GB production PostgreSQL database from a physical server to Azure Database for PostgreSQL, with the application staying live.

**What went wrong (this is the interesting part):**
- `pg_dump`/`pg_restore` version mismatch between source (PG17) and initial target tooling (PG14) — dumps silently failed with cryptic "unsupported archive version" errors
- SSL connections dropped mid-transfer on the largest table (12GB) — traced to Burstable-tier CPU credit exhaustion, not network issues
- Ran out of headroom on a 32GB storage allocation restoring a database that needed ~25GB with index overhead

**What I did:**
- Installed matching major-version PostgreSQL client tools
- Used `screen` sessions with TCP keepalive tuning to make long-running transfers resilient to SSH disconnects
- Temporarily scaled the target to General Purpose tier for the restore, then scaled back down post-migration
- Verified integrity via table counts, row counts, and size comparisons before cutover

**Why this is a good interview answer:** shows methodical debugging under pressure, understanding of database internals (not just running commands), and cost-conscious infrastructure decisions (scale up only when needed).

---

## Story 2: The production incident (good "tell me about a time something broke" answer)

**Situation:** After deploying 7 microservices, the whole environment became unreachable — even admin database connections were being rejected.

**Root cause:** `max_connections` limit (50, inherited from an earlier Burstable tier) was exhausted by 7 services each opening HikariCP's default pool size (~10 connections each = 70+ connections).

**What I did:**
- Diagnosed via `pg_stat_activity` even though I initially couldn't connect (used the reserved superuser slot logic to understand why even `psql` was locked out)
- Applied a two-layer fix: capped each service's connection pool size (`spring.datasource.hikari.maximum-pool-size`), and raised the database's `max_connections` limit for headroom
- Restarted services one at a time, verifying connection count stayed healthy after each, rather than restarting all 7 at once and re-triggering the same failure

**Why this is a good interview answer:** demonstrates incident response under a self-inflicted outage, not panicking, and fixing both the symptom and the root cause rather than just the symptom.

---

## Story 3: The dependency archaeology (good "describe a hard debugging problem" answer)

**Situation:** One of 7 microservices (`bseapi`) consistently failed to build with "could not resolve CommonLibrary:0.0.2" — a version that appeared to not exist anywhere.

**What I did:**
- Established that 6 services needed `CommonLibrary:0.0.1` (Java 21/Spring Boot 3.4) while `bseapi` needed `0.0.2` — genuinely different, not a typo
- Found via `pom.xml` inspection that `bseapi` had commented-out lines showing someone had previously *attempted* the Java 21 upgrade and reverted — a clue that 0.0.2 was tied to a specific, older stack
- Discovered (by asking) an undocumented git branch, `release_jdk8`, which built the correct Java 8-compatible version of the shared library
- Checked out that branch, built and installed the correct artifact, resolving the build cleanly

**Why this is a good interview answer:** shows persistence through ambiguity, reading code for historical clues (the commented-out lines), and knowing when to ask a clarifying question versus keep digging alone.

---

## Story 4: Untangling a live git incident (good "describe working with a difficult process/team situation" answer)

**Situation:** A fix I needed (a missing `staticwebapp.config.json` file causing all client-side routes to 404) kept disappearing after every push — I'd fix it, deploy successfully, and it would vanish again.

**What I did:**
- Used `git log`, `git show`, and branch comparison to trace that a teammate was **force-pushing** to the same shared branch concurrently, silently discarding my commits
- Diagnosed the actual React/Azure Static Web Apps quirk underneath it all: the config file needs to live in `public/`, not the project root, for Create React App to include it in the build output — a subtlety that had caused multiple "successful" deployments to silently exclude the fix
- Recognized this had become a coordination problem, not a technical one, and flagged the need to sync with the teammate rather than keep re-fixing in a loop

**Why this is a good interview answer:** shows git fluency beyond basic commands (using history inspection to diagnose team behavior, not just your own mistakes), and the judgment to recognize when a "technical" problem is actually a communication problem.

---

## Story 5: The JDK version fleet inconsistency (good "attention to detail" answer)

**Situation:** Found, while checking a duplicated set of services on the physical server, that a production service (`bseapi`) had been silently running on JDK 21 instead of its required JDK 8 since a systemd misconfiguration weeks earlier — undetected because it happened to still run without crashing.

**What I did:**
- Used `/proc/<pid>/exe` to definitively verify which JDK binary a running process actually used (more reliable than trusting `ExecStart` config, which can silently resolve to a different binary via `PATH`)
- Systematically checked every service across two parallel deployments, not just the one that was actively broken
- Fixed by always pinning the full explicit JDK path in service startup commands, never relying on the ambiguous `java` command

**Why this is a good interview answer:** shows proactive investigation beyond what was asked, and a durable fix (explicit paths) rather than a one-time patch.

---

## Skills demonstrated (for resume bullets / skills section)

**Cloud infrastructure:** Azure (VMs, VNets, NSGs, Managed PostgreSQL, Static Web Apps, disk snapshotting strategy for portability)
**Systems administration:** Linux, systemd, nginx reverse proxy design, SSH key management, disk partitioning
**Databases:** PostgreSQL migration, connection pool tuning, backup/restore under production constraints
**Application deployment:** Java/Spring Boot, Maven dependency resolution across multiple JDK versions, CI/CD via GitHub Actions
**Debugging methodology:** systematic hypothesis elimination (branch → pipeline → cache → root cause), reading `/proc` for ground-truth process info, git history archaeology
**Frontend/backend integration:** CORS configuration, SPA routing, CDN caching behavior (CloudFront), mixed-content/HTTPS issues

---

## Honest reflection points (good for "what would you do differently" questions)

- **Documentation gap:** the `release_jdk8` branch and the CommonLibrary versioning split cost real time because it wasn't documented anywhere discoverable — in hindsight, I'd push for a README in that repo explaining the two-version strategy
- **Branch protection:** the repeated force-push collisions on `deploy_to_finesse_uat` point to a process gap — that branch would benefit from branch protection rules or at least a "who's working on this right now" convention
- **Config-as-code earlier:** a lot of today's manual VM/nginx/systemd setup would have been faster and safer as Terraform from the start — worth advocating for on the next environment build

---

## Quick technical talking points (for rapid-fire technical questions)

- **Why separate OS disk from data disk?** Disposability — the OS can be rebuilt from a base image; the data disk can be snapshotted independently and reattached to a fresh VM, giving a clean disaster-recovery story without needing full VM-level backups
- **Why HikariCP pool caps matter?** Each service's default pool size multiplies across every instance — 7 services × default 10 connections easily exceeds a modest database's connection limit even under light traffic
- **Why `/proc/<pid>/exe` over `java -version`?** `java -version` only tells you what the *current shell's* PATH resolves to — it says nothing about what a *specific already-running process* is actually using
- **Why CORS breaks when moving frontend to a CDN?** Once frontend and backend are on different origins (even subdomains), every API call becomes cross-origin, requiring explicit `Access-Control-Allow-Origin` handling server-side — no way around it, by design
