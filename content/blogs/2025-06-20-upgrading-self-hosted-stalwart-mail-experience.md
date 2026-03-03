---
title: "Upgrading Self Hosted Stalwart Mail Experience"
date: 2025-06-20T08:45:32+07:00
tags: ["stalwart mail", "self hosted", "upgrade"]
draft: false
---

## Steps

1. Get the current version of Stalwart Mail.
   - current version: v0.8.0
2. Check the documentation for the latest version.
   - upgrading documentation: <https://github.com/stalwartlabs/stalwart/blob/main/UPGRADING.md>
3. Upgrade to the next major version incrementally
   - next major version: v0.9.x
   - next major version: v0.10.x
   - next major version: v0.11.x
   - next major version: v0.12.x
4. Fix issues that arise during the upgrade process.
   - issue: failed to start when using v0.11.0
     root causes:
     - <https://github.com/stalwartlabs/stalwart/issues/1057>
     - <https://github.com/stalwartlabs/stalwart/issues/1104>
       resolution: use v0.11.3 when upgrading from v0.10 to v0.11
   - issue: cannot login to web admin after upgrading it to latest version when
     using the old version of stalwart mail
     root causes:
     - <https://www.reddit.com/r/stalwartlabs/comments/1ikfz8u/error_trying_to_log_in_admin/>
       resolution: finish the upgrade to the latest version v0.12.4
5. Export and backup the data before upgrading.
   - export and backup data using stalwart mail CLI command
   - backup the database data using the database management tool

### Get the current version of Stalwart Mail

Check the docker-compose file to get the current version of Stalwart Mail.

### Check the documentation for the latest version

Read the upgrading documentation to understand the changes and steps required
for each version upgrade.

### Upgrade to the next major version incrementally

Change the stalwart mail docker image tag in the docker-compose file to:

1. v0.9.0
1. v0.10.0
1. v0.11.3 (see the issue above)
1. v0.12.4 (see the issue above)

### Fix issues that arise during the upgrade process

1. Failed to start when using v0.11.0
   - This issue is caused by missing code in the v0.11.0 release.
   - The solution is to use v0.11.3 instead of v0.11.0.
2. Cannot login to web admin after upgrading it to the latest version when
   still using the old version of stalwart mail.
   - Upgraded the web admin using the web admin UI or API.
     ```bash
     curl -k -u admin:your-password https://localhost/api/update/webadmin
     ```
   - This issue is caused by a change in the web admin version.
   - The solution is to finish the upgrade to the latest version v0.12.4.

### Export and backup the data before upgrading

1. Export and backup data using stalwart mail CLI command
   - Use the `stalwart mail export` command to export the data.
   - Save the exported data to a safe location.
2. Backup the database data using the database management tool
   - Use the database management tool to create a backup of the database.
   - Save the backup to a safe location.
