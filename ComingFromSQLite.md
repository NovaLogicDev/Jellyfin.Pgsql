# Jellyfin PostgreSQL Plugin

This plugin adds PostgreSQL support to the Jellyfin server, replacing the default SQLite database.

> [!WARNING]
> **HIGHLY EXPERIMENTAL**
> This plugin runs on unstable builds of Jellyfin (10.11+). Use at your own risk and **always backup your data** before attempting a migration.

> [This README is autogen from Google Gemini Pro, Cir. Dec 2025, Human review and revision conducted by NovalogicDev]

## Quick Start (Fresh Install)

The container automatically configures the database connection using environment variables.

1.  **Update `docker-compose.yaml`**:
    Use the PostgreSQL image and set the required connection variables.

    ```yaml
    services:
      jellyfin:
        image: ghcr.io/novalogicdev/jellyfin.pgsql:latest # -YOU WILL WANT TO CHANGE THIS
        volumes:
          - /path/to/config:/config
          - /path/to/cache:/cache
          - /path/to/media:/media
        environment:
          - POSTGRES_HOST=postgres
          - POSTGRES_PORT=5432
          - POSTGRES_DB=jellyfin
          - POSTGRES_USER=jellyfin
          - POSTGRES_PASSWORD=jellyfin
          # Optional SSL settings:
          # - POSTGRES_SSLMODE=Require
          # - POSTGRES_TRUSTSERVERCERTIFICATE=true
        depends_on:
          postgres:
            condition: service_healthy

      postgres:
        image: postgres:18
        environment:
          - POSTGRES_USER=jellyfin
          - POSTGRES_PASSWORD=jellyfin
          - POSTGRES_DB=jellyfin
        volumes:
          - ./postgres-data:/var/lib/postgresql/data
        healthcheck:
          test: ["CMD-SHELL", "pg_isready -U jellyfin"]
          interval: 10s
          timeout: 5s
          retries: 5
    ```

2.  **Start the Container**:
    The entrypoint script will automatically generate the correct `database.xml` and apply necessary EF migrations.

---

## Migration Guide: Moving from SQLite

Migrating an existing Jellyfin instance (running on SQLite) to this PostgreSQL build requires specific steps to avoid startup crashes. The container cannot automatically upgrade "Legacy" configuration files, so you must clear specific files to trigger the container's self-repair logic.

### 1. Prepare Configuration
Before starting the new container, perform the following "Surgical Deletions" in your mapped `/config` folder.

#### A. Trigger Database Re-Configuration
The container fails to start if it detects an old `database.xml` configured for SQLite.
* **Action:** Delete `config/database.xml`.
* **Result:** The container detects the missing file and generates a new one with your `POSTGRES_*` environment variables.

#### B. Fix Network Configuration Crash
Legacy `network.xml` files often contain schema data incompatible with 10.11, causing `InvalidOperationException` crashes.
* **Action:** Delete `config/network.xml`.
* **Result:** Jellyfin generates a valid default network configuration on boot.

#### C. Reset Migration State
If the server crashed during a previous attempt, `migrations.xml` may be in a corrupt/pending state.
* **Action:** Delete `config/migrations.xml`.
* **Result:** Forces Jellyfin to re-scan the database state.

### 2. The "Fresh Install" Trick (Crucial)
Some hardcoded maintenance tasks (like `RemoveDuplicateExtras`) attempt to read the SQLite `library.db` even when running on Postgres, causing a crash. You must trick Jellyfin into skipping these checks.

* **Action:** Edit `config/system.xml`.
* **Change:** Find `<IsStartupWizardCompleted>` and set it to `false`.
    ```xml
    <IsStartupWizardCompleted>false</IsStartupWizardCompleted>
    ```
### 2. The "Startup Toggle" Trick (Crucial)
Some hardcoded maintenance tasks (like `RemoveDuplicateExtras`) attempt to read the SQLite `library.db` even when running on Postgres, causing a crash. You must perform a "Toggle" sequence to bypass this.

**Phase 1: Bypass the Crash**
1.  Edit `config/system.xml` and change `<IsStartupWizardCompleted>` to `false`.
2.  Start the container.
3.  **Wait** until the server fully boots and is accessible via the Web UI (you will see the Setup Wizard).
    * *Note:* Do **not** attempt to complete the wizard; it will likely get stuck because your users already exist in the database.

**Phase 2: Restore Normal Operation**
1.  Stop the container.
2.  Edit `config/system.xml` and change `<IsStartupWizardCompleted>` back to `true`.
3.  Start the container again.
    * *Result:* The legacy maintenance tasks are now marked as "skipped" by the previous boot, and the server loads your existing libraries and users normally.

### 3. Data Migration (SQLite -> Postgres)
The container includes `pgloader` and attempts to migrate your data automatically if conditions are met.

**If the auto-migration fails or is skipped:**
You can manually trigger the migration from inside the container using the included load script.

```bash
docker exec -it jellyfin pgloader /jellyfin-pgsql/jellyfindb.load
```
(Note: Ensure your old `jellyfin.db` is present in the data directory before running this.)

# Development & Building
The project includes a `Dockerfile` that handles the build, plugin installation, and entrypoint setup.

## Build Container:

```bash
docker build -t jellyfin-pgsql -f docker/Dockerfile .
```

## Build Migrations:

### Add migration
Run the following to add a new migration for efcore
```bash
dotnet ef migrations add {MIGRATION_NAME} --project "/workspaces/Jellyfin.Pgsql/Jellyfin.Plugin.Pgsql" -- --migration-provider Jellyfin-PgSql
```

then bundle the migrations into an idempotent sql script

```bash
dotnet ef migrations script --idempotent --output ./docker/PGSqlMigrate.sql --project "/workspaces/Jellyfin.Pgsql/Jellyfin Plugin.Pgsql" --  --migration-provider Jellyfin-PgSql
```
