# Building and Deploying Applications on Coolify

**Mandatory policy.** v0.3 draft. Owner: IT / Infrastructure.

| Part | For | When |
|---|---|---|
| 1. Rules | Everyone | Read once |
| 2. New application setup | IT | Once per app |
| 3. Building | Whoever is writing the code | Read once, then daily |
| 4. Into testing | Automatic | Every change |
| 5. Testing to production | IT | Every release |
| 6. Pilot | IT | Once |

**If you're here to write code, you only need Part 3.**

---

# Part 1: Rules

**Production credentials never leave the server.** We don't put a production password, connection string or service role key on a laptop, in a `.env`, in a chat message or in a repo. If one gets out, we rotate it.

**Each application is one Coolify project with four resources:**

| Resource | Purpose | Branch |
|---|---|---|
| Supabase dev | Database and login for local work **and** testing | - |
| Supabase prod | Live database. Real data. | - |
| App testing | Testing address | `develop` |
| App production | Live address | `main` |

Both app containers stay running, on separate addresses. We don't switch one off to bring the other up.

**One Supabase stack serves one application.** Self-hosted Supabase is single-project by design. One stack is one database, and the login service assumes it's serving one app. So every app gets its own pair. There's no sensible way around this.

**What moves between environments:** code, and schema as `.sql` files in `supabase/migrations/`. Both go through GitHub.

**What never moves:** data, user accounts, secrets. Each stack keeps its own. That means a login created in testing won't exist in production, which is expected.

**Non-negotiable:**

1. Every repo has a `CLAUDE.md`. See 2.7.
2. Schema changes happen by adding a migration file, and no other way. **Not by clicking around in Supabase Studio.** A change made that way isn't recorded anywhere, so production never picks it up and the app breaks when we release.
3. Lockfiles get committed.
4. New dependencies are named and justified in the PR.
5. No secrets in the repo, including in history.
6. Nothing goes to production without running on testing first.

**Why we're on Coolify.** Our data stays on infrastructure we own and administer. The cost is fixed rather than per-seat. The build is defined in the repo, so environments match. In return we've taken on uptime, patching and DR ourselves, which sits with IT rather than with whoever is writing the app.

---

# Part 2: New application setup

**IT. Once per app, about 45 minutes.**

Pick a short lowercase name, e.g. `client-notes`. Addresses follow this pattern:

```
client-notes.internal.conscious.co.uk              prod app
client-notes-testing.internal.conscious.co.uk      testing app
db-client-notes.internal.conscious.co.uk           prod Supabase
db-client-notes-dev.internal.conscious.co.uk       dev Supabase
```

**2.1 Repo.** Private GitHub repo with `main` and `develop`. Set `develop` as the default so new work lands there rather than in production. Protect `main` so changes only arrive by pull request.

**2.2 Coolify project.** Named after the app, with all four resources inside it.

**2.3 Supabase prod.** Add resource, pick the Supabase service template, name it `db-<app>`, set the domain and deploy. Put the anon key, service role key and Postgres connection string in the password manager. Keep Studio on the internal network only.

**2.4 Supabase dev.** Same again as `db-<app>-dev`. The anon key from this one is the only Supabase credential the developer gets.

**2.5 App testing.** Add resource, private repo via deploy key. Add the key Coolify generates to GitHub under Deploy keys, read-only. Branch `develop`, build pack **Dockerfile** rather than Nixpacks, testing domain, auto-deploy on push enabled. Variables:

```
NEXT_PUBLIC_SUPABASE_URL=https://db-<app>-dev.internal.conscious.co.uk
NEXT_PUBLIC_SUPABASE_ANON_KEY=<dev anon key>
SUPABASE_SERVICE_ROLE_KEY=<dev service role key>
NODE_ENV=production
```

**2.6 App production.** Same again on `main`, with the prod domain and prod keys. **Leave auto-deploy off,** so releases are something we choose to do.

**2.7 Template repo.** We build this once and reuse it for every app. It's what takes the decisions a non-developer can't make out of the picture.

```
CLAUDE.md  Dockerfile  .dockerignore  .env.example
.gitignore  README.md  package.json  supabase/migrations/  src/
```

Check `.gitignore` covers `.env` and `node_modules` before anyone uses it.

`CLAUDE.md` is how we get the rules followed, because Claude reads it and sticks to it without the developer having to remember any of it:

```markdown
# Project rules

## Stack
Next.js (App Router), TypeScript, Supabase (self-hosted), Tailwind.
Do not introduce another framework, ORM or state library.

## Database
- Change the database only by adding a numbered .sql file to
  supabase/migrations/. Never tell the user to edit it in Studio.
- Migrations are forward-only. Correct a mistake with a new migration.
- Prefer additive changes. Add a column, backfill, drop the old one in a
  later release. Never rename or drop in the same migration as the code
  change that depends on it.
- Every table has row level security enabled with an explicit policy.
  A table without one is a data leak.
- Reference and lookup data goes in a migration as inserts, not a seed
  file. Seed files never run in production.

## Secrets
- Configuration comes from environment variables only.
- Never commit .env or print a key to the terminal.
- The service role key bypasses all security: server-side only, never in
  code that runs in the browser.

## Dependencies
- Prefer the framework and standard library.
- Before adding a package, tell the user its name, purpose, and why
  nothing installed will do. Wait for agreement.
- Never add a package to work around an error you have not diagnosed.

## Environment variables
NEXT_PUBLIC_* is visible to the browser and must use the public Supabase
address. Server-only variables are never prefixed NEXT_PUBLIC_.

## Working with the user
Assume no programming knowledge. Say what you are about to do in plain
language first. Never ask the user to edit a file by hand. Give exact
commands and say what they will do.
```

**2.8 Handover.** The developer gets the repo and a Contributor invite, the **dev** Supabase address and anon key, and Part 3. Nothing from 2.3.

---

# Part 3: Building an application

**You don't need to know how to program. Claude writes the code. Your job is to describe what you want, check it works, and pass it on.**

**3.1 Install these once:** Node.js LTS from nodejs.org, Git from git-scm.com, the Claude Code desktop app, and GitHub Desktop from desktop.github.com. Restart afterwards.

**3.2 Clone the project.** Sign in to GitHub Desktop and clone the repo IT gave you. Make a note of the folder it goes into.

**3.3 Open it.** Point Claude Code at that folder.

**3.4 First-time setup.** Ask Claude:

> Set up this project for local development. My Supabase URL is [address] and my anon key is [key]. Create the .env file, install dependencies, and start the development server.

It'll tell you when the app is running, usually at `http://localhost:3000`. Open that in your browser. **If this doesn't work, stop and ask IT.** Don't spend time trying to fix it.

**3.5 Build it.** Describe what you want, and be specific:

> Add a page listing every note belonging to the logged-in user, newest first, with a button to add one.

The page in your browser updates as Claude works. Have a look at it, and if it isn't right, say what's different from what you wanted.

If Claude asks to add a package it'll explain why. If the reason doesn't make sense to you, say no and ask what else could be done instead.

When Claude changes the database it writes a file into `supabase/migrations/`. Don't delete these. They're how your change gets to the live app later.

**3.6 Check it.** Does the new thing work when you click it? Does everything that worked before still work? Log out and back in, and check again.

**3.7 Send it on.**

> Commit these changes and push to the develop branch, with a message describing what changed.

The testing address updates within a few minutes. Check your change there as well, because it's closer to the real thing than your computer is. Then let IT know.

**3.8 Picking it up the next day.** "Pull the latest changes and start the development server." That's the whole routine.

**3.9 Don't** change anything in Supabase Studio, paste keys or passwords into a chat, or work on `main`. If Claude offers to do any of these, say no.

---

# Part 4: Into testing

Push to `develop`, the webhook fires, testing rebuilds in a few minutes, and the developer checks it at the testing address.

**Still to confirm:** whether a new migration gets applied to the dev Supabase automatically on deploy or needs doing by hand. Step 4 of the pilot answers this, and we rewrite this section once we know.

**If the build fails,** Coolify shows the log. It's usually a package installed locally but not committed, or a variable the code expects that Coolify doesn't have.

---

# Part 5: Testing to production

**IT only.**

**5.1 Before we promote.** The change has been seen working at the testing address, not just on someone's laptop. Migration files have been read, checking for `DROP`, `TRUNCATE` and `ALTER COLUMN`. The lockfile is committed and matches `package.json`. New dependencies are justified. Any new table has an RLS policy.

**5.2 Promote.** PR from `develop` to `main`, review, merge. **Apply migrations to production before deploying the code** (5.3), then trigger the production deploy by hand and check it. The order matters, because doing it the other way round leaves the live app running against a schema it doesn't recognise.

**5.3 Migration script.** Run from a checkout of `main` on the server or a jump host, not from a developer's machine. The connection string comes from the secret store.

```bash
#!/usr/bin/env bash
set -euo pipefail

# Applies pending Supabase migrations. Requires PROD_DB_URL.

if [ -z "${PROD_DB_URL:-}" ]; then
  echo "PROD_DB_URL is not set. Aborting."
  exit 1
fi

if [ ! -d supabase/migrations ]; then
  echo "No supabase/migrations directory. Aborting."
  exit 1
fi

echo "Pending:"
supabase migration list --db-url "$PROD_DB_URL"

echo "Applying."
supabase db push --db-url "$PROD_DB_URL"

echo "Done."
```

For Jenkins, take the credentials from the store and fail the pipeline on a non-zero exit. **We need to check the CLI invocation against our Supabase version during the pilot,** as it hasn't been tested against our instance yet.

**5.4 If it goes wrong.** A code problem we roll back in Coolify. A migration problem has no undo, so we write a corrective migration forward. If data has been destroyed we restore from the host backup and lose everything since the snapshot, which is what 5.1 is there to avoid.

---

# Part 6: Pilot

A personal notes list. Log in by email, see your own notes only, add and delete them. Small, but it covers login, a table, RLS, and a migration reaching production.

1. Work through Part 2 for `notes-pilot`, noting anything that didn't match this document.
2. Build the template repo (2.7) as we go. That's the reusable part.
3. Follow Part 3 as though we didn't know anything, using only the dev anon key. Anywhere we fall back on knowledge a non-developer wouldn't have, that's a gap in Part 3.
4. Get Claude to add the notes table via a migration, with RLS.
5. Push to `develop` and confirm testing updates on its own.
6. Add a second feature that needs a schema change, say a "pinned" flag. That gives us two migrations, which is the real test.
7. Promote it through Part 5, script included.
8. Log in on production. There should be no notes there, because production has no data. That's the design working as intended.

**Things to watch:** whether dev migrations apply on their own; whether the browser can reach Supabase, because if server-side works and the browser doesn't then `NEXT_PUBLIC_SUPABASE_URL` is wrong; and whether the containers trust ConPKI certificates, since Node ignores the system trust store and needs `NODE_EXTRA_CA_CERTS`.

---

# Open items

1. Do dev migrations apply automatically on a testing deploy? (Part 4)
2. Check the Supabase CLI invocation in 5.3.
3. Confirm the Supabase template behaves as described on Coolify v4.1.2.
4. Should `NODE_EXTRA_CA_CERTS` be set on every app resource as standard?
5. Memory sizing for two Supabase stacks per app, and how we place containers across VMs.
6. Effective date, and who signs off exceptions.
