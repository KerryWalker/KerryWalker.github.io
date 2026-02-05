---
layout: post
title: Migrating from MyTeslaMate to a Local TeslaMate Installation on Home Assistant
excerpt: How to export your database from MyTeslaMate and restore it to a self-hosted TeslaMate instance running on Home Assistant OS.
tags:
  - home-assistant
  - tesla
  - teslamate
---

I've been using [MyTeslaMate](https://myteslamate.com) — a hosted TeslaMate service — to track my Tesla's drives, charges, and efficiency. It's a great service, but I wanted to move to a self-hosted setup running on my Home Assistant machine. The main challenge: migrating all my historical data.

## The Setup

- **Source:** MyTeslaMate (hosted service)
- **Destination:** TeslaMate add-on on Home Assistant OS
- **Database:** PostgreSQL 17 add-on
- **Tools:** pgAdmin4 add-on, Samba share

## Step 1: Install the Required Add-ons

In Home Assistant, install these add-ons from the Add-on Store:

- **TeslaMate** (from the community add-on repository)
- **PostgreSQL 17** (or similar PostgreSQL add-on)
- **pgAdmin4** (for database management)
- **Samba share** (for file transfer)

Get TeslaMate running first with a fresh database to confirm everything works, then we'll replace the database with your backup.

## Step 2: Download Your Backup from MyTeslaMate

MyTeslaMate provides a backup API endpoint. Your backup URL will look something like:

```
https://app.myteslamate.com/backup/api?token=your-token-here
```

You can find this in your MyTeslaMate account settings. Download the file — it'll be a gzipped SQL dump.

## Step 3: Prepare the Backup File

Extract the gzip file to get the raw SQL:

```bash
gunzip backup.gz
```

Here's the important bit: **the backup will have all database objects owned by `myteslamate`**, and it also includes grants to your personal MyTeslaMate user account. Your local PostgreSQL won't have these users, so we need to replace them.

Open the SQL file in a text editor and do two find-and-replace operations:

**Replace 1 — Database owner:**
- Find: `myteslamate`
- Replace: `postgres`

**Replace 2 — Your MyTeslaMate user account:**

Look for a username in the format `user_xxxxxxxx` near the end of the file (in the GRANT statements). This is your personal MyTeslaMate username — the part after `user_` is your account identifier.

- Find: `user_xxxxxxxx` (your specific username, e.g., `user_tyrbxifpm18`)
- Replace: `postgres`

Save the file. This ensures all tables, functions, and permissions will be owned by your local database user, and removes references to users that don't exist locally.

## Step 4: Transfer the Backup to Home Assistant

Copy the modified SQL file to your Home Assistant's `/share` folder via the Samba share. On Windows, navigate to `\\homeassistant.local\share` (or use your HA's IP address) and drop the file there.

## Step 5: Prepare the Database

Open pgAdmin4 (click "Open Web UI" on the add-on) and connect to your PostgreSQL server using the credentials from your PostgreSQL add-on configuration.

Run these SQL commands in the Query Tool:

```sql
-- Drop the existing teslamate database if it exists
DROP DATABASE IF EXISTS teslamate;

-- Create a fresh database
CREATE DATABASE teslamate;
```

Now connect to the new `teslamate` database and create the required extensions:

```sql
\c teslamate

CREATE EXTENSION IF NOT EXISTS cube;
CREATE EXTENSION IF NOT EXISTS earthdistance;
```

## Step 6: Restore the Backup

In pgAdmin4, with the `teslamate` database selected, open the Query Tool and run:

```sql
\i /share/your-backup-file.sql
```

Replace `your-backup-file.sql` with your actual filename.

The restore will take a minute or two depending on how much data you have. You'll likely see one error at the very end:

```
error: invalid command \unrestrict
```

This is fine — it's a MyTeslaMate-specific command that PostgreSQL doesn't recognise. Your data is all there.

## Step 7: Verify the Data

Run some quick queries to confirm your data imported correctly:

```sql
SELECT COUNT(*) FROM public.positions;
SELECT COUNT(*) FROM public.drives;
SELECT MIN(start_date), MAX(start_date) FROM public.drives;
```

You should see your full history — positions, drives, and the date range of your data.

## Step 8: Restart TeslaMate

Go to **Settings → Add-ons → TeslaMate** and restart the add-on. It should now connect to the database and show all your historical data in the Grafana dashboards.

## Troubleshooting

### Missing extensions error

If the restore fails with errors about `cube` or `earthdistance`, make sure you created those extensions before running the restore.

### Wrong database user

Check your TeslaMate add-on configuration — it needs to match the database credentials. The add-on config should have:

- **Database host:** Your PostgreSQL hostname (e.g., `db21ed7f-postgres-latest`)
- **Database name:** `teslamate`
- **Database user:** `postgres` (or whatever user you're using)
- **Database password:** Your PostgreSQL password

## Summary

The key steps are:

1. Download your backup from MyTeslaMate
2. Replace `myteslamate` with `postgres` in the SQL file
3. Create the required extensions (`cube`, `earthdistance`)
4. Restore using `\i /share/backup.sql` in pgAdmin
5. Ignore the `\unrestrict` error at the end
6. Restart TeslaMate

Once migrated, you've got full local control of your Tesla data — and you can integrate it with Home Assistant automations, custom Grafana dashboards, and whatever else you want to build.