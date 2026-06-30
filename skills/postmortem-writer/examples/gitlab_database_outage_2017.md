# Incident Postmortem: GitLab.com Database Outage (January 31, 2017)

**Date of Report:** 2017-02-01
**Incident Date:** 2017-01-31
**Severity:** Critical (Outage)
**Total Duration:** ~18 hours
**Lead Investigators:** Infrastructure Engineering Team

---

## 1. Executive Summary
On January 31, 2017, GitLab.com experienced a critical service outage due to the accidental deletion of the primary database directory during database replication replication recovery. Widespread failures in the backup and restore verification pipeline resulted in an extended recovery process. Data was ultimately restored to a point-in-time prior to the incident using an LVM snapshot taken six hours before the deletion, resulting in the loss of approximately six hours of user data (including issues, merge requests, users, and comments).

* **Start Time (T0):** 2017-01-31 17:20 UTC (Staging maintenance start; database load issues started at 19:00 UTC)
* **Deletion Event:** 2017-01-31 23:30 UTC
* **Detection Time:** 2017-01-31 23:30 UTC (Immediate by operator)
* **Mitigation Started:** 2017-01-31 23:35 UTC (Outage page displayed, backup retrieval initiated)
* **Full Recovery Time:** 2017-02-01 17:30 UTC

---

## 2. Customer & Business Impact
* **User-Facing Symptoms:** GitLab.com was completely unavailable; users encountered 502/503 errors and outage information landing pages.
* **Affected Services/Dependencies:** Web application, API, Git-over-SSH/HTTP, background runners, and issue tracking.
* **Data Loss:** Approximately 6 hours of database writes (issues, merge requests, comments, user accounts, and project creation). Git repositories themselves were stored on separate NFS servers and remained intact.
* **Quantitative Metrics:** 100% service outage for ~18 hours; millions of developers unable to access their code repositories or collaborate.

---

## 3. Timeline (Key Events)

| Timestamp (UTC) | Event / Action / Decision |
| :--- | :--- |
| **2017-01-31 17:20** | **T0 (Context):** Staging setup begins. Operator takes a manual LVM snapshot of the production DB to refresh staging database. |
| **19:00** | GitLab.com experiences database load spikes (suspected spam and a hard-delete cascade of an employee account flagged for abuse). User actions degrade. |
| **23:00** | Due to load, PostgreSQL secondary replication lag exceeds threshold. Replication fails as required WAL segments are recycled on primary before secondary syncs. |
| **23:10** | Operator attempts secondary resynchronization. Operator wipes data directory on secondary and runs `pg_basebackup`. The utility hangs silently. |
| **23:15** | To troubleshoot the hang, operator increases `max_wal_senders` on primary. PostgreSQL fails to restart due to semaphore limits (`max_connections` was set to 8000). Operator reduces connections to 2000; DB restarts successfully. |
| **23:25** | Operator attempts `pg_basebackup` again, running it with `strace`. The utility continues to poll silently (waiting for primary replication data connection, which was undocumented behavior). |
| **23:30** | **Trigger Event:** Operator assumes the secondary's data directory holds locking state from previous failed attempts. Operator runs command to wipe data directory, but does so on the **primary** instead of the secondary. |
| **23:30** | Operator realizes the error and terminates the deletion process. However, ~300 GB of data had already been deleted from the primary cluster. |
| **23:35** | Outage page is deployed. Disaster recovery procedures are initiated. |
| **2017-02-01 00:00 - 03:00** | Engineers search for database backups. Multiple restore procedures fail due to misconfigurations (see Section 4). |
| **03:00** | Engineers locate the manual LVM snapshot taken at 17:20 UTC on January 31. Decision made to restore staging data copy back to production. |
| **17:30** | Staging snapshot successfully restored to production cluster, replication re-established, and GitLab.com brought back online. |

---

## 4. Root Cause Analysis

### The Trigger
An operator executed a recursive directory removal command (`rm -rf`) on the primary database server instead of the secondary replica due to lack of visual distinction between terminal sessions.

### Systemic Causes
* **Lack of Safe Tooling / Automation for Resyncing Replica:** Replica resynchronization was a manual, error-prone process requiring directory deletion and command execution, rather than an automated script or runbook validation check.
* **Terminal Interface Ambiguity:** No safeguards, environment warning headers, or execution prompts distinguished the primary database shell from the secondary database shell.

### Why Defenses and Recovery Safeguards Failed
1. **`pg_dump` Backups Failed:** The automated daily backup script used a default PostgreSQL client version (9.2) that was incompatible with the server's PostgreSQL version (9.6). The command errored out silently.
2. **Notification Failures:** Cron error emails were rejected by external mail receivers because the server sending the alerts lacked proper SPF/DMARC configuration.
3. **Azure Disk Snapshots Failed:** Azure snapshots were configured for NFS filesystems, but not for the PostgreSQL database server disks.
4. **AWS S3 Backup Bucket Empty:** The `pg_dump` backup storage bucket was empty because the upload stage of the script was never reached due to the silent version incompatibility crash.
5. **LVM Auto-Snapshots Silently Disabled:** Automated LVM snapshots were disabled on the DB servers due to historical performance issues, but this status was not tracked.

---

## 5. Detection & Response Analysis
* **Detection:** The deletion event was detected immediately by the operator executing the command, who saw the shell prompt paths and cancelled the process.
* **Alert Effectiveness:** Alerting systems fired immediately after the database went offline, but were secondary to human detection. Backups monitoring alerts did not exist, leading to silent failure of backups for months.
* **Runbook & Tooling Adequacy:** The database runbooks were out of date and lacked instructions explaining that `pg_basebackup` hangs silently while waiting for primary WAL streaming. This directly prompted the operator to suspect directory pollution and attempt the wipe.

---

## 6. Lessons Learned

### What Went Well
* The manual LVM snapshot taken by the operator prior to staging maintenance served as the sole savior of the data, saving GitLab.com from permanent data loss.
* The response coordination on Slack and live streaming of the recovery process fostered trust and transparency with the developer community.

### Gaps & What Went Wrong
* **Undocumented Tools Behavior:** Lack of documentation regarding `pg_basebackup` silent polling led to operational confusion.
* **Failure to Test Restores:** Backup procedures were never verified with regular automated restores, allowing version mismatch errors to persist undetected.
* **Email Alert Routing Gaps:** Silent DMARC rejects of error alerts broke the operational feedback loop.

---

## 7. Action Items

| Action Item | Owner | Priority | Target Date | Verifiable Outcome / Goal |
| :--- | :--- | :--- | :--- | :--- |
| [ ] Correct `pg_dump` client version mismatch in backup scripts. | Database Infrastructure Team | High | Resolved | Backups run without version error. |
| [ ] Configure automated restore testing of S3 backups to staging. | Infrastructure Team | High | 2017-02-15 | Automated cron restores backup to staging DB daily and verifies row counts. |
| [ ] Implement shell/terminal prompt color distinction (red for production). | Security & Operations Team | Medium | 2017-02-05 | Production SSH prompts show warning headers and custom red background. |
| [ ] Fix DMARC and SPF email configurations for alert domains. | IT Systems Team | High | 2017-02-08 | Automated alerts reach designated inbox without bounces. |
| [ ] Re-evaluate LVM snapshot automation and performance overhead. | DB Team | Medium | 2017-02-28 | LVM snapshots automated daily during low-traffic windows. |
