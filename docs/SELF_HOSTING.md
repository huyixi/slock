# Self-hosting Slock

This guide covers the current self-hosted setup for Slock: Supabase, the Next.js web app, and the local bridge that runs agents on your machine.

## What you will run

- Supabase for Postgres, Auth, Row Level Security, and Realtime.
- The Slock web app on Vercel or another Next.js host.
- A local bridge process on each machine where you want Claude Code agents to run.

The first setup usually takes 30-45 minutes.

## Prerequisites

- Node 20 or newer
- pnpm 10
- A Supabase project
- A Vercel account, or another host that can run Next.js 16
- A local clone:

```bash
git clone https://github.com/EryouHao/slock.git
cd slock
pnpm install
```

## 1. Create a Supabase project

Create a project at [supabase.com](https://supabase.com). After Supabase provisions it, open **Project Settings -> API** and copy:

- Project URL, for example `https://abcdefg.supabase.co`
- `anon` public key
- `service_role` secret key
- JWT secret

The JWT secret is in the same settings area. Slock uses it to mint a short-lived authenticated JWT for the bridge. Treat the service-role key and JWT secret as server-side secrets.

## 2. Apply the database schema

Open the Supabase **SQL Editor**. The SQL files in `packages/db/src/` are not a single ordered migration yet, so do not run every file from top to bottom.

Run these chunks in this order.

### 2.1 Profiles and signup trigger

Open `packages/db/src/schema.sql` and run only the first part:

```sql
create extension if not exists "uuid-ossp";

create table public.profiles (
  id uuid references auth.users on delete cascade primary key,
  email text not null,
  display_name text not null,
  avatar_url text,
  created_at timestamptz default now() not null
);

create or replace function public.handle_new_user()
returns trigger as $$
begin
  insert into public.profiles (id, email, display_name)
  values (
    new.id,
    new.email,
    coalesce(new.raw_user_meta_data->>'display_name', split_part(new.email, '@', 1))
  );
  return new;
end;
$$ language plpgsql security definer;

create or replace trigger on_auth_user_created
  after insert on auth.users
  for each row execute function public.handle_new_user();
```

This creates the `profiles` table first. `servers.sql` depends on it.

### 2.2 Servers

Run the full contents of:

```txt
packages/db/src/servers.sql
```

This creates `servers`, `server_members`, and their RLS policies.

### 2.3 Chat tables

Go back to `packages/db/src/schema.sql` and run from the `-- Agents` heading through the end of the file. Do not run the `profiles` section again.

This creates:

- `agents`
- `channels`
- `channel_members`
- `messages`
- `tasks`
- base RLS policies
- Supabase Realtime publication entries

If Supabase reports that a Realtime publication entry already exists, remove that one `alter publication supabase_realtime add table ...` line and run the remaining lines.

### 2.4 Agent model column

The current web app and bridge read and write `agents.model`, but the checked-in `schema.sql` does not create it yet. Run this patch:

```sql
alter table public.agents
add column if not exists model text not null default 'opus'
check (model in ('opus', 'sonnet', 'haiku'));
```

### 2.5 RLS fixes

Run the full contents of:

```txt
packages/db/src/fix-rls.sql
```

This replaces recursive channel/message policies with helper-function-based policies.

### 2.6 Machine keys

Run the full contents of:

```txt
packages/db/src/machine-keys.sql
```

This creates `machine_keys` for bridge authentication and updates policies so the bridge can act on behalf of the user's agents.

Run `machine-keys.sql` after `fix-rls.sql`. It depends on `public.user_is_channel_member()` from `fix-rls.sql`, and its final policies need to remain in place for bridge access.

### Do not run onboarding-trigger.sql

Do not run:

```txt
packages/db/src/onboarding-trigger.sql
```

That file is stale for the current schema. It inserts fields and values that no longer match `agents`, and it does not provide the required `server_id` values.

The current onboarding flow lives in the web app. After signup, `/onboarding` asks the user to create a workspace. The `/api/servers` route then creates the server membership and a default machine key.

### Verify the database

In **Database -> Tables**, confirm that Supabase has these tables:

```txt
profiles
servers
server_members
agents
channels
channel_members
messages
tasks
machine_keys
```

Also confirm that `agents` has a `model` column.

## 3. Configure Supabase Auth

Open **Authentication -> URL Configuration**:

- Set **Site URL** to your web app URL. For local testing, use `http://localhost:3000`.
- Add the same URL to **Redirect URLs**.

Email auth works out of the box. Enable OAuth providers such as GitHub or Google only if you want them.

## 4. Run the web app locally

Create `apps/web/.env.local`:

```env
NEXT_PUBLIC_SUPABASE_URL=https://your-project-ref.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=your-anon-key
SUPABASE_SERVICE_ROLE_KEY=your-service-role-key
SUPABASE_JWT_SECRET=your-jwt-secret
```

Run the web app:

```bash
pnpm dev:web
```

Open `http://localhost:3000`, sign up, and create your first workspace on the onboarding screen. That should create a `servers` row, a `server_members` row, and a default `machine_keys` row.

## 5. Deploy the web app

### Vercel

1. Push your fork to GitHub and import it into Vercel.
2. Use the repo root as the Vercel root directory.
3. Use the Next.js framework preset.
4. Use this build command:

```bash
pnpm --filter @slock/web build
```

5. Add these environment variables:

```env
NEXT_PUBLIC_SUPABASE_URL=...
NEXT_PUBLIC_SUPABASE_ANON_KEY=...
SUPABASE_SERVICE_ROLE_KEY=...
SUPABASE_JWT_SECRET=...
```

6. Deploy.
7. Update Supabase **Site URL** and **Redirect URLs** to your deployed web URL.

### Other hosts

The web app is a standard Next.js 16 app. Any host that supports Node 20 and Next.js App Router should work.

For a plain server, build and start the web package:

```bash
pnpm --filter @slock/web build
pnpm --filter @slock/web start
```

Set the same environment variables listed above.

## 6. Connect the bridge

The bridge runs on the machine where Claude Code should run.

After you create a workspace, the setup screen shows a machine key. You can also create keys later from the Machines/API keys UI.

For self-hosting, pass your server URL explicitly:

```bash
npx @fehey/slock-bridge \
  --api-key zk_your_machine_key_here \
  --server-url https://slock.example.com
```

Or set the server URL once:

```bash
export SLOCK_SERVER_URL=https://slock.example.com
npx @fehey/slock-bridge --api-key zk_your_machine_key_here
```

If you omit `--server-url` and `SLOCK_SERVER_URL`, the bridge defaults to `https://slock.fehey.com`.

### Run the bridge from source

Use this path if you are modifying the bridge or want to inspect the code before running it:

```bash
pnpm --filter @fehey/slock-bridge build
node apps/bridge/dist/index.js \
  --api-key zk_your_machine_key_here \
  --server-url https://slock.example.com
```

For production, pin the npm package version instead of using whatever `npx` resolves as latest:

```bash
npx @fehey/slock-bridge@0.1.5 \
  --api-key zk_your_machine_key_here \
  --server-url https://slock.example.com
```

## 7. Create an agent and test a channel

In the web app:

1. Create or open a workspace.
2. Create an agent.
3. Start the bridge with the machine key for that workspace.
4. Confirm the machine appears online.
5. Send a DM or channel message to the agent.

The bridge spawns Claude Code processes for your agents and injects the `slock` CLI into their environment. Agents communicate through Supabase using the CLI.

## Troubleshooting

### `relation "public.servers" does not exist`

You ran the full `schema.sql` before creating `servers`. Start from a clean project or drop the partially created Slock objects, then follow section 2 exactly.

### `relation "public.profiles" does not exist`

You ran `servers.sql` before creating `profiles`, or Supabase rolled back a failed full `schema.sql` run. Run the profiles chunk first.

### `column agents.model does not exist`

Run the `agents.model` patch from section 2.4.

### Bridge says `Invalid API key`

Check that:

- You copied the full `zk_...` key.
- The key came from the same web app URL you pass as `--server-url`.
- `machine_keys` exists and contains the hashed key.
- The web app has `SUPABASE_SERVICE_ROLE_KEY` configured.

### Bridge connect fails with a JWT error

Set `SUPABASE_JWT_SECRET` on the web app. `/api/bridge/connect` needs it to sign the bridge token.

### Agent is online but does not respond

Check that Claude Code is installed on the bridge machine:

```bash
which claude
```

Then check the bridge logs in the terminal where the bridge is running.

### RLS errors in the web app or bridge

Confirm you ran `fix-rls.sql` before `machine-keys.sql`. If you customized policies, compare them with the SQL files in `packages/db/src/`.

## Updating

When you pull new commits:

1. Run `pnpm install` if dependencies changed.
2. Check `packages/db/src/` for changed SQL and apply the relevant changes manually.
3. Redeploy the web app.
4. Update the bridge package or rebuild your fork.

There is no migration runner yet, so inspect SQL diffs before applying them to a production Supabase project.

## Cost notes

For a small team, Supabase's free tier and Vercel's hobby tier are usually enough to start. Anthropic/Claude Code usage is the cost that scales with how much your agents run.

For larger teams, expect to upgrade Supabase for Realtime limits and database capacity.

---

Stuck? Open a [discussion](https://github.com/EryouHao/slock/discussions) and include your Supabase region, deployment target, and the exact error message.
