# Supabase Activity Monitor

Prevent your Supabase projects from pausing due to inactivity by periodically inserting, monitoring, and deleting entries in specified tables across multiple Supabase databases.

## What It Does

Supabase pauses free-tier projects after 7 days of inactivity. This tool keeps your databases active by:

1. Inserting a random string into a specified table for each configured database
2. Monitoring the number of entries in that table
3. Automatically deleting a random entry if the table exceeds 10 records
4. Logging successes and failures, and generating a detailed status report

The script runs automatically via GitHub Actions on a daily scheduled cron job, completely serverless and free.

## Features

- Support for multiple Supabase databases via a single `config.json` file
- Environment variable-based API key management for security
- Automatic table entry cleanup to prevent table bloat
- Detailed status reporting and failure logging
- Opt-in workflow activation (disabled by default for safety)

## Project Structure

```
.
├── .github/workflows/keep-alive.yml  # GitHub Actions workflow
├── main.py                           # Main execution script
├── services/
│   └── supabase_service.py           # Supabase client wrapper
├── helpers/
│   └── utils.py                      # Utility functions
├── config.json                       # Database configuration (required)
└── README.md                         # This file
```

## Setup

### Prerequisites

- A GitHub account
- One or more Supabase projects (free tier or paid)
- A `service_records` table created in each Supabase database (see [Database Setup](#database-setup))

### Step 1: Fork This Repository

Fork this repository to your GitHub account.

### Step 2: Create Your Configuration File

Create a new file named `config.json` in the repository root with your Supabase project details:

```json
[
	{
		"name": "MyProject",
		"supabase_url": "https://your-project-id.supabase.co",
		"supabase_key_env": "SUPABASE_KEY_1",
		"table_name": "service_records"
	}
]
```

**Important:** Always use `supabase_key_env` (not `supabase_key`) to reference environment variables. This keeps your API keys out of the codebase.

### Step 3: Commit and Push

Add and commit your `config.json` to the repository:

```bash
git add config.json
git commit -m "Add Supabase activity monitor configuration"
git push origin main
```

### Step 4: Configure GitHub Secrets and Variables

Navigate to your forked repository on GitHub and go to **Settings** > **Secrets and variables** > **Actions**.

#### Enable the Workflow (Variables Tab)

1. Click the **Variables** tab
2. Click **New repository variable**
3. Name: `ENABLE_GITHUB_ACTIONS`
4. Value: `true`

This opt-in mechanism prevents the workflow from running unintentionally on forks.

#### Add Your API Keys (Secrets Tab)

1. Click the **Secrets** tab
2. Click **New repository secret**
3. Add a secret for each database, matching the environment variable names in your `config.json`:
   - Name: `SUPABASE_KEY_1`
   - Value: Your Supabase `anon` or `service_role` API key
4. Repeat for all databases (`SUPABASE_KEY_2`, `SUPABASE_KEY_3`, etc.)

**Security Note:** Use your `service_role` key if Row Level Security (RLS) is enabled on your `service_records` table. The `anon` key will fail if RLS policies block inserts. See [Database Setup](#database-setup) for more details.

### Step 5: Enable GitHub Actions

1. Go to the **Actions** tab in your forked repository
2. Click "I understand my workflows, go ahead and enable them"

### Step 6: Test the Workflow

1. In the Actions tab, select "Supabase Activity Monitor"
2. Click **Run workflow** to execute it manually
3. Review the logs to verify everything works correctly

**That's it!** The workflow will automatically run daily at midnight UTC.

## Configuration Reference

The `config.json` file accepts an array of database objects with the following properties:

| Property           | Required | Description                                                                   |
| ------------------ | -------- | ----------------------------------------------------------------------------- |
| `name`             | Yes      | A friendly name for the database, used in logs and reports.                   |
| `supabase_url`     | Yes      | The Supabase project URL (e.g., `https://xyz.supabase.co`).                   |
| `supabase_key_env` | No\*     | The name of the environment variable containing the API key. **Recommended.** |
| `supabase_key`     | No\*     | A hardcoded API key string. **Not recommended.**                              |
| `table_name`       | No       | The table to interact with. Defaults to `service_records`.                    |

\*You must provide either `supabase_key_env` or `supabase_key`. Environment variables are strongly preferred for security.

**Example with multiple databases:**

```json
[
	{
		"name": "ProductionDB",
		"supabase_url": "https://prod-project.supabase.co",
		"supabase_key_env": "SUPABASE_PROD_KEY",
		"table_name": "service_records"
	},
	{
		"name": "StagingDB",
		"supabase_url": "https://staging-project.supabase.co",
		"supabase_key_env": "SUPABASE_STAGING_KEY",
		"table_name": "service_records"
	}
]
```

## Customizing the Schedule

By default, the workflow runs daily at midnight UTC. To change this schedule, edit `.github/workflows/keep-alive.yml` and modify the cron expression:

```yaml
schedule:
  - cron: "0 0 * * *" # Daily at midnight UTC
```

Use [crontab.guru](https://crontab.guru) to generate custom cron expressions. For example:

- Every day at midnight: `0 0 * * *`
- Every Monday at 9 AM UTC: `0 9 * * 1`
- Every 6 hours: `0 */6 * * *`

## Database Setup

This project requires a target table in your Supabase Postgres database. The default table name is `service_records`, but you can configure any table name in `config.json`.

### SQL to Create the Table

Run this SQL in the Supabase SQL Editor:

```sql
CREATE TABLE IF NOT EXISTS "service_records" (
  id BIGINT GENERATED BY DEFAULT AS IDENTITY,
  name TEXT NULL DEFAULT ''::TEXT,
  random UUID NULL DEFAULT gen_random_uuid(),
  CONSTRAINT "service_records_pkey" PRIMARY KEY (id)
);

-- Optional: Add initial placeholder data
INSERT INTO "service_records" (name)
VALUES ('placeholder'), ('example')
ON CONFLICT DO NOTHING;
```

### Row Level Security (RLS) Considerations

If your Supabase project has Row Level Security (RLS) enabled by default for new tables, you must either:

1. **Disable RLS** on the `service_records` table (if it contains no sensitive data):

   ```sql
   ALTER TABLE "service_records" DISABLE ROW LEVEL SECURITY;
   ```

2. **Create a permissive policy** for the `service_role` key:

   ```sql
   CREATE POLICY "Allow service_role full access" ON "service_records"
   FOR ALL USING (true) WITH CHECK (true);
   ```

3. **Use the `service_role` API key** in your GitHub Secrets. The `anon` key typically cannot bypass RLS unless specific policies are created for it.

## Security Best Practices

- **Never commit API keys** to your repository. Always use GitHub Secrets via `supabase_key_env`.
- **Use the `service_role` key** if your table has RLS enabled, as it bypasses all policies.
- **Keep your `config.json` minimal.** Only include non-sensitive configuration data.
- **Review your fork's visibility.** If your repository is public, ensure no secrets are accidentally exposed in logs.

## How It Works

1. **GitHub Actions Trigger:** The workflow runs on the schedule defined in `keep-alive.yml`, or manually via the Actions tab.
2. **Environment Setup:** GitHub Actions checks out your code, sets up Python 3.12, and installs the `supabase` package.
3. **Configuration Loading:** `main.py` reads `config.json` and resolves API keys from the environment variables defined in `supabase_key_env`.
4. **Database Iteration:** For each database in the config:
   - A random 10-character string is generated using `secrets` (cryptographically secure).
   - The string is inserted into the specified table via the Supabase REST API.
   - The total row count is fetched.
   - If the count exceeds 10, a random row is deleted to prevent table bloat.
5. **Reporting:** A detailed status report is logged, including insert success, row count, and deletion status for each database.

## Troubleshooting

### Workflow Does Not Run

- Verify the `ENABLE_GITHUB_ACTIONS` repository variable is set to `true`.
- Check the **Actions** tab to ensure workflows are enabled for the repository.

### "Configuration file 'config.json' not found"

- Ensure you created `config.json` and committed it to the repository.
- The file must be in the repository root.

### "Supabase URL or Key missing"

- Verify that `supabase_url` and `supabase_key_env` are correctly set in `config.json`.
- Ensure the corresponding secret exists in GitHub Secrets and matches the name in `supabase_key_env` exactly.

### Insert or Delete Failures (403 / 401 Errors)

- Confirm you are using the correct API key.
- If RLS is enabled on your table, use the `service_role` key or create an appropriate RLS policy.
- Check that the table name in `config.json` exactly matches the table name in your database (case-sensitive for quoted identifiers).

### Table Bloat (Too Many Rows)

- The script deletes a random entry only when the count exceeds 10.
- If you see many more rows, check the workflow logs for deletion errors.

## Contributing

Contributions are welcome. Feel free to open an issue or submit a pull request if you have suggestions, bug fixes, or improvements.

## License

This project is licensed under the MIT License. See the `LICENSE` file for details.
