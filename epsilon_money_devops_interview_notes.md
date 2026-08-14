# DevOps / Cloud Migration Project Notes — Epsilon Money UAT

## 1. Project Context

This project involved moving the UAT frontend delivery from Azure Static Web Apps / Azure CI-CD to AWS S3 + CloudFront while keeping the existing GitHub branch strategy.

Key environment components I worked with:

- Frontend: React client portal
- Git branch used for UAT deployment: `deploy_to_finesse_uat`
- Frontend build output: React `build/`
- AWS deployment target: S3 bucket
- CDN: AWS CloudFront
- Backend/API: Linux VM with Nginx + Spring Boot services
- Database environments:
  - Azure Database for PostgreSQL — UAT baseline
  - Physical PostgreSQL server — local/on-prem UAT copy
- CI/CD: GitHub Actions
- Application runtime: Spring Boot, Java, systemd user services
- Reverse proxy: Nginx

---

## 2. Frontend CI/CD Migration: Azure → AWS

### What I changed

The frontend was previously deployed through Azure Static Web Apps with Azure CI/CD.

I removed the Azure CI/CD pipeline and added a GitHub Actions workflow for AWS deployment using the same UAT branch.

Workflow file:

`client-portal/.github/workflows/deploy_client_portal.yml`

Important deployment flow:

```text
Developer pushes to deploy_to_finesse_uat
              |
              v
        GitHub Actions
              |
              v
          npm install
              |
              v
          npm build
              |
              v
        AWS S3 bucket
              |
              v
        CloudFront CDN
              |
              v
        UAT frontend
```

The workflow performs:

1. Checkout source code.
2. Install/use Node.js 18.
3. Run `npm install`.
4. Run the React build.
5. Configure AWS credentials.
6. Sync the React `build` directory to S3.
7. Create a CloudFront invalidation for `/*`.

Example commands used by the workflow:

```bash
npm install
npm run build --if-present

aws s3 sync build s3://${{ vars.AWS_CLIENT_PORTAL_FINESSE }}

aws cloudfront create-invalidation \
  --distribution-id ${{ vars.CDN_CLIENT_PORTAL_FINESSE }} \
  --paths "/*"
```

### Important interview point

I did not create a new Git branch just because the deployment platform changed.

The deployment branch remained:

```text
deploy_to_finesse_uat
```

The CI/CD mechanism changed from Azure to AWS.

---

## 3. Frontend Environment Configuration

The React `.env` contained the backend API base URL.

Old/commented endpoint:

```text
http://ec2-18-61-144-185.ap-south-2.compute.amazonaws.com:8089/loginService/
```

Active endpoint:

```text
https://api.ans2.epsilonmoney.com/loginService/
```

This is important because moving the frontend from Azure to CloudFront does not automatically move or change the backend.

Architecture concept:

```text
ans2.epsilonmoney.com
        |
        v
AWS CloudFront
        |
        v
S3 React frontend
        |
        | HTTPS API calls
        v
api.ans2.epsilonmoney.com
        |
        v
Nginx
        |
        v
Spring Boot backend services
```

---

## 4. CORS / CloudFront Testing

After moving the frontend to CloudFront, an API request was tested against:

```text
https://api.ans3.epsilonmoney.com/loginService/v1/auth/data?dataType=country
```

The request included:

```text
Origin: https://d8zc9giq2mcdr.cloudfront.net
Access-Control-Request-Method: GET
Access-Control-Request-Headers: content-type,authorization
```

The response was:

```text
HTTP/2 502
server: nginx/1.24.0 (Ubuntu)
```

### What this told us

The immediate problem was not simply "CloudFront is broken."

A `502` from Nginx indicated that Nginx was unable to successfully reach the upstream backend.

Nginx configuration showed:

```text
proxy_pass http://localhost:9010;
```

So the investigation moved to the backend service listening on port `9010`.

---

## 5. Nginx → Spring Boot Backend Investigation

I searched the server configuration:

```bash
sudo grep -RIn "9010" /etc/nginx /home/devops/launch2 /home/devops/.config/systemd/user 2>/dev/null
```

It showed:

```text
/etc/nginx/sites-available/api.ans3.epsilonmoney.com
/etc/nginx/sites-enabled/api.ans3.epsilonmoney.com
/home/devops/launch2/aggregator/application.properties
```

The application configuration contained:

```text
server.port=9010
```

This established the expected chain:

```text
Nginx :443
   |
   v
localhost:9010
   |
   v
aggregator.jar / Spring Boot
```

---

## 6. Spring Boot Service Failure

The systemd service was:

```text
epsilonmoney2-aggregator.service
```

Initial status showed it was inactive/failed.

The service definition was inspected with:

```bash
systemctl --user cat epsilonmoney2-aggregator.service
```

Important configuration:

```ini
[Service]
Type=simple
Environment="SPRING_CONFIG_LOCATION=/home/devops/launch2/aggregator/application.properties"
ExecStart=/usr/lib/jvm/java-25-openjdk-amd64/bin/java -jar /home/devops/launch2/aggregator/aggregator.jar
Restart=always
RestartSec=5
```

It also had dependencies on:

```text
epsilonmoney2-bseapi.service
epsilonmoney2-client.service
epsilonmoney2-notifications.service
epsilonmoney2-refdata.service
epsilonmoney2-support.service
```

### Root cause found

Spring Boot failed during startup because Log4j2 could not find its configuration file.

Error:

```text
Could not initialize Log4J2 logging from
file:/epsilonmoney/logs2/EMLoginService_log4j.xml
```

The deeper cause was:

```text
java.io.FileNotFoundException:
/epsilonmoney/logs2/EMLoginService_log4j.xml
(No such file or directory)
```

The file was later confirmed to exist:

```bash
ls -lb /epsilonmoney/logs2/ | grep EMLogin
```

Result:

```text
-rw-rw-r-- 1 devops devops 1392 ... EMLoginService_log4j.xml
```

This showed an important troubleshooting lesson:

> A file can exist when checked manually but still fail during application startup if the effective runtime configuration, dependency ordering, environment, path, permissions, or process context is different.

The application was then able to run manually, but it initially stopped when the MobaXterm session closed.

---

## 7. systemd and MobaXterm Lesson

The backend was manually started during troubleshooting.

When the process was attached to the MobaXterm terminal/session, closing MobaXterm caused the process to stop.

This reinforced the importance of running production/UAT backend processes under systemd rather than directly from an interactive shell.

Target model:

```text
systemd
  |
  +--> aggregator
  +--> bseapi
  +--> client
  +--> notifications
  +--> refdata
  +--> reporting
  +--> support
```

Current service check used:

```bash
systemctl --user list-units --type=service | grep epsilonmoney2
```

The services were subsequently shown as:

```text
epsilonmoney2-aggregator.service       active running
epsilonmoney2-bseapi.service           active running
epsilonmoney2-client.service           active running
epsilonmoney2-notifications.service    active running
epsilonmoney2-refdata.service          active running
epsilonmoney2-reporting.service        active running
epsilonmoney2-support.service          active running
```

### Interview point

I can explain why systemd is preferable to starting a Java process manually:

- Process lifecycle management
- Automatic restart
- Dependency ordering
- Centralized logs
- Startup at login/boot depending on configuration
- Reduced dependency on an SSH/MobaXterm session

---

# 8. Database Migration / Comparison

The UAT database situation was investigated because the physical UAT PostgreSQL database had been copied from Azure earlier, but the Azure database could have received changes during the following week.

Therefore, Azure UAT was selected as the desired/current baseline.

The two environments were:

```text
Azure PostgreSQL UAT
        |
        | desired/current baseline
        v
Physical UAT PostgreSQL
```

A database list on Azure showed:

```text
Epsilonmoney
azure_maintenance
azure_sys
postgres
template0
template1
```

The relevant database was:

```text
Epsilonmoney
```

The physical PostgreSQL server contained:

```text
Epsilonmoney
Epsilonmoney_finesse
postgres
template0
template1
```

The physical UAT database of interest was:

```text
Epsilonmoney_finesse
```

---

## 9. Table Count Comparison

The total table count was approximately 206 tables.

Instead of manually comparing every table, exact row counts were generated for both environments.

The comparison command was:

```bash
diff -u /tmp/physical_exact_counts.csv /tmp/azure_exact_counts.csv
```

The result identified tables whose row counts differed.

Examples:

```text
emm_accord_absolute_return_master
physical = 11368
azure    = 11273
difference = 95
```

```text
emm_accord_nav
physical = 8578795
azure    = 8517261
difference = 61534
```

```text
mst_amfi_nav
physical = 2935769
azure    = 2900549
difference = 35220
```

There were also tables where Azure had MORE rows than physical:

```text
emm_bse_client
physical = 57
azure    = 59
difference = -2
```

```text
emm_staging_finesse_clients
physical = 696
azure    = 1116
difference = -420
```

This demonstrated that the databases were not identical.

---

# 10. Database Comparison Analysis

The row-count differences fell into several categories.

### A. Large reference/master data differences

Examples:

- `emm_accord_nav`
- `mst_amfi_nav`
- `emm_accord_portfolio_sectorwiseholding_master`
- `emm_accord_mfportfolio_master`
- `emm_accord_schememonthwise_expense_ratio`

These differences can be caused by:

- Data refresh timing
- Master-data updates
- Incremental loads
- Deleted/expired records
- Different dump dates

These should not automatically be "fixed" by copying rows blindly.

### B. Client/UAT transactional data differences

Examples:

- `emm_client`
- `emm_client_address`
- `emm_client_asset`
- `emm_client_bank_account`
- `emm_client_goal_asset_mapping`
- `emm_user`
- `emm_user_roles`

These are more sensitive.

Before changing them, identify whether Azure or physical is the source of truth for UAT and whether the physical environment has any new UAT transactions that must be preserved.

### C. Audit/log tables

Examples:

- `emm_response_audit`
- `emm_schedule_job_audit`

These can naturally differ because applications continue writing to them.

### D. Staging tables

Example:

```text
emm_staging_finesse_clients
physical = 696
azure    = 1116
```

Staging tables can have large differences without meaning that the core application data is wrong.

---

# 11. Important Decision Made

Because the requirement was:

> Azure UAT should be the base.

The correct migration strategy is NOT:

```text
Blindly merge physical + Azure
```

Instead:

```text
Azure UAT
   |
   | authoritative baseline
   v
Physical UAT
```

The safest approach is:

1. Take a backup of the current physical UAT database.
2. Confirm the Azure UAT database is the authoritative source.
3. Restore/copy Azure UAT data to the physical UAT database.
4. Verify schema/table counts.
5. Verify application connectivity.
6. Start all backend services.
7. Test APIs.
8. Test the frontend through CloudFront.
9. Validate critical UAT workflows.

---

# 12. PostgreSQL Connectivity Troubleshooting

An external application attempted to connect to:

```text
jdbc:postgresql://122.179.152.105:5432/Epsilonmoney_finesse
```

but received:

```text
java.net.SocketTimeoutException: Connect timed out
```

The question was whether PostgreSQL port 5432 was externally exposed.

On the physical server, I checked:

```bash
sudo ss -ltnp | grep 5432
```

Output:

```text
LISTEN 0 780 0.0.0.0:5432 0.0.0.0:* users:(("postgres",pid=1532,fd=6))
LISTEN 0 780 [::]:5432 [::]:* users:(("postgres",pid=1532,fd=7))
```

This means PostgreSQL was listening on all IPv4 and IPv6 interfaces, not only localhost.

UFW also showed:

```text
5432/tcp ALLOW IN Anywhere
```

I also checked iptables:

```bash
sudo iptables -L INPUT -n -v | grep 5432
```

No matching rule was returned.

The server's public IP was checked with:

```bash
curl -4 ifconfig.me
```

Result:

```text
122.179.156.105
```

Important finding:

The external client was trying:

```text
122.179.152.105
```

while the server reported its public IP as:

```text
122.179.156.105
```

That mismatch is a major troubleshooting lead.

However, listening on `0.0.0.0:5432` and allowing UFW does NOT guarantee internet reachability.

Other possible blocking points include:

- Cloud/provider firewall
- Router/NAT
- Corporate network firewall
- ISP firewall
- Security appliance
- Incorrect public IP
- PostgreSQL `pg_hba.conf`
- Network ACL/security policy

---

# 13. Key Commands I Learned / Used

## Check listening ports

```bash
sudo ss -ltnp
```

Specific port:

```bash
sudo ss -ltnp | grep 5432
```

## Check UFW

```bash
sudo ufw status
```

## Check iptables

```bash
sudo iptables -L INPUT -n -v
```

## Check systemd user services

```bash
systemctl --user list-units --type=service
```

## Inspect service configuration

```bash
systemctl --user cat epsilonmoney2-aggregator.service
```

## Inspect service status

```bash
systemctl --user status epsilonmoney2-aggregator.service
```

## Follow service logs

```bash
journalctl --user -u epsilonmoney2-aggregator.service -f
```

## Search configuration

```bash
sudo grep -RIn "9010" /etc/nginx /home/devops/launch2 /home/devops/.config/systemd/user 2>/dev/null
```

## Find Log4j configuration files

```bash
sudo find /home/devops/launch2 /epsilonmoney /home/devops/launch \
-name '*log4j*.xml' 2>/dev/null
```

## Check public IP

```bash
curl -4 ifconfig.me
```

## PostgreSQL table list

```sql
SELECT table_schema, table_name
FROM information_schema.tables
WHERE table_schema NOT IN ('pg_catalog', 'information_schema')
AND table_type = 'BASE TABLE'
ORDER BY table_schema, table_name;
```

---

# 14. Interview Explanation — 60 Second Version

"I worked on a UAT deployment migration where the frontend delivery was moved from Azure Static Web Apps to AWS S3 and CloudFront. I kept the existing GitHub deployment branch and changed the GitHub Actions workflow so that the React application was built and synchronized to S3, followed by a CloudFront cache invalidation.

During testing I investigated API 502 and CORS-related issues. I traced the request through CloudFront, Nginx and the Spring Boot backend. Nginx was proxying to localhost port 9010, so I checked the corresponding systemd service and application logs. The Spring Boot service was failing during startup because of a Log4j2 configuration file issue. I also converted the backend processes from manual execution to systemd-managed services so they would not depend on an SSH or MobaXterm session.

I also compared the Azure PostgreSQL UAT database with the physical UAT PostgreSQL database. Instead of manually checking around 206 tables, I generated exact row counts and used diff to identify differences. Since Azure UAT was defined as the authoritative baseline, the next step was to back up the physical database and synchronize it from Azure rather than blindly merging the differences.

Finally, I investigated external PostgreSQL connectivity by checking the listening address, UFW, iptables and public IP. PostgreSQL was listening on 0.0.0.0:5432 and UFW allowed 5432, but the client was trying a different public IP than the server reported, which became the next network troubleshooting point."

---

# 15. Interview Topics to Focus On

## Priority 1 — AWS deployment

Be comfortable explaining:

- S3 static website hosting
- CloudFront
- Origin
- Cache invalidation
- HTTPS/custom domains
- DNS
- GitHub Actions
- AWS credentials/secrets
- IAM
- CI/CD pipeline flow

You should be able to draw:

```text
GitHub
   |
   v
GitHub Actions
   |
   v
React Build
   |
   v
S3
   |
   v
CloudFront
   |
   v
Custom Domain
```

## Priority 2 — Linux troubleshooting

Know:

```bash
ss
netstat
curl
grep
find
ps
systemctl
journalctl
df
du
top
free
```

Understand ports, processes, permissions, services and logs.

## Priority 3 — Nginx

Know:

- Reverse proxy
- `proxy_pass`
- HTTP vs HTTPS
- TLS termination
- 502 Bad Gateway
- upstream connectivity
- access/error logs
- virtual hosts/server blocks

Typical troubleshooting chain:

```text
Client
  ↓
DNS
  ↓
Nginx :443
  ↓
proxy_pass
  ↓
Spring Boot :9010
```

## Priority 4 — PostgreSQL

Focus on:

- Databases vs schemas vs tables
- Users/roles
- `pg_hba.conf`
- `listen_addresses`
- Port 5432
- Backups/restores
- `pg_dump`
- `pg_restore`
- Row counts
- Basic SQL
- Connection troubleshooting

## Priority 5 — Networking

Understand:

- Public/private IP
- NAT
- Firewall
- Security groups
- UFW
- iptables
- DNS
- TCP connection
- Ports
- TLS
- CORS
- HTTP status codes

Especially understand the difference between:

```text
Port is listening
```

and:

```text
Port is reachable from the internet
```

They are NOT the same thing.

---

# 16. Strong Troubleshooting Method for Interviews

When asked "How would you troubleshoot a 502?", explain:

```text
1. Check DNS
2. Check whether Nginx is running
3. Check Nginx access/error logs
4. Check proxy_pass target
5. Check whether backend port is listening
6. Check backend systemd status
7. Check application logs
8. Curl the backend locally
9. Curl the public API
10. Check firewall/network rules
11. Check application dependencies
12. Retest end-to-end
```

For database connectivity:

```text
1. Verify hostname/IP
2. Verify port
3. Check PostgreSQL listening address
4. Check firewall
5. Check pg_hba.conf
6. Test locally
7. Test from remote host
8. Check network/NAT/security rules
9. Check credentials/database name
10. Check SSL requirements
```

---

# 17. Security Lessons

Do NOT put real passwords, private keys, AWS access keys, database credentials or tokens into GitHub.

For GitHub Actions:

Use:

```text
GitHub Secrets
GitHub Variables
OIDC / IAM roles where possible
```

For databases:

- Restrict 5432 to trusted source IPs.
- Prefer VPN/private networking over exposing PostgreSQL directly to the internet.
- Use TLS.
- Use least-privilege database users.
- Rotate credentials if exposed.

For this project, never commit a real database password such as one shared through chat/messages.

---

# 18. What I Should Be Able to Explain Without Looking at Notes

Before an interview, practice these questions:

### CI/CD

1. What happens when code is pushed to `deploy_to_finesse_uat`?
2. Why use GitHub Actions?
3. Why S3 + CloudFront?
4. Why invalidate CloudFront?
5. Where should AWS credentials be stored?
6. How would you roll back a bad frontend deployment?

### Linux

1. How do you check whether port 9010 is listening?
2. How do you check why a service failed?
3. How do you view live logs?
4. Why does a manually started Java process stop when SSH/MobaXterm closes?
5. How does systemd solve this?

### Nginx

1. What does `proxy_pass` do?
2. What causes a 502?
3. How would you trace Nginx → Spring Boot?
4. What is the difference between 502 and 504?

### PostgreSQL

1. How do you check whether PostgreSQL is listening?
2. What is `pg_hba.conf`?
3. What is the difference between `listen_addresses` and `pg_hba.conf`?
4. How do you back up a PostgreSQL database?
5. How do you restore it?
6. How do you compare two databases?

### Networking

1. What is the difference between private and public IP?
2. What is NAT?
3. Why can `ss` show port 5432 listening while an external client still gets a timeout?
4. What is CORS?
5. What does HTTP 502 mean?

---

# 19. Project Skills Demonstrated

This project gave hands-on exposure to:

- AWS S3
- AWS CloudFront
- GitHub Actions
- CI/CD
- React deployment
- Linux administration
- Nginx
- Spring Boot
- Java
- systemd
- PostgreSQL
- Database backup/restore
- Database comparison
- DNS/API troubleshooting
- CORS troubleshooting
- HTTP troubleshooting
- Firewall troubleshooting
- Public/private networking
- Production-style incident troubleshooting

---

# 20. Final Interview Positioning

Do not describe the work as only:

> "I deployed a React application."

Describe it as:

> "I worked on an end-to-end UAT deployment migration and troubleshooting workflow involving GitHub Actions, AWS S3, CloudFront, DNS/API endpoints, Nginx reverse proxying, Spring Boot services managed through systemd, PostgreSQL database synchronization, and Linux/network troubleshooting."

The strongest part of the experience is the ability to **trace a problem across layers**:

```text
Browser
  ↓
CloudFront
  ↓
DNS
  ↓
Nginx
  ↓
Spring Boot
  ↓
PostgreSQL
  ↓
Network / Firewall
```

That end-to-end troubleshooting mindset is what to emphasize in the interview.
