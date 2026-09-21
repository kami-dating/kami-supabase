# kami-supabase

Supabase backend for Kami, a free, ad-free dating app for sapphic/WLW people.

There are two hosted Supabase projects: **preview** and **production**.

## Branches and deployment

| Branch | Deploys to | Trigger |
|---|---|---|
| `staging` | preview project | push to `staging` |
| `main` | production project | push to `main` (PRs only, always from `staging`) |

Work happens on `staging`. A PR from `staging` into `main` promotes it to production.

Deployment uses Supabase's GitHub integration, not GitHub Actions, so there are no tokens or
environment variables to maintain in GitHub. Each project is connected to this repo in the
Supabase dashboard (Project Settings → Integrations → GitHub):

| Project | Production branch | Deploy to production |
|---|---|---|
| preview | `staging` | on |
| production | `main` | on |

- The Supabase directory path is `.` (the `supabase/` folder is at the repo root).
- Leave Branching off. Do not add a workflow that also runs `db push`, or migrations would be
  applied twice.
- The integration applies new migrations (and Edge Functions and Storage buckets declared in
  `config.toml`). It ignores Auth settings and seed files.
- Consider requiring the Supabase check on PRs into `main`, so a PR whose migrations fail
  cannot be merged.

Never change the schema in the Supabase dashboard on preview or production.

## Local development

Requires [Docker](https://docs.docker.com/get-docker/) and the [Supabase CLI](https://supabase.com/docs/guides/local-development/cli/getting-started).

```sh
supabase start      # run Supabase locally
supabase db reset   # apply migrations and seed files
supabase stop
```

## Changing the database

The schema is declared in `supabase/schemas/*.sql`. Edit those files, then generate a migration
from the diff, review it, and commit it in `supabase/migrations/`. Data changes (inserts,
updates) are not captured by the diff and need hand-written migrations.

## Dev admin account

`supabase db reset` seeds a local-only admin (`admin@kami.lgbt`) from `supabase/seed.admin.sql`.
That file is gitignored because it contains a password hash. To create it:

1. Copy `supabase/seed.admin.sql.example` to `supabase/seed.admin.sql`.
2. Generate a bcrypt hash of a **throwaway** password (not one you use anywhere else):

   ```powershell
   $d = New-Item -ItemType Directory -Path (Join-Path $env:TEMP "kami-hash") -Force
   Set-Location $d
   npm init -y | Out-Null
   npm i bcryptjs
   $env:PW = Read-Host "Password" -MaskInput
   node -e "console.log(require('bcryptjs').hashSync(process.env.PW, 10))"
   Remove-Item Env:PW
   ```

3. Replace `PASTE_BCRYPT_HASH_HERE` in `seed.admin.sql` with the printed hash.

Seed files never run on preview or production. Create the real admin accounts there through
the Supabase Auth dashboard or admin API.
