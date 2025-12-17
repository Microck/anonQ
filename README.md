<p align="center">
  <a href="https://github.com/Microck/anonq">
    <img src="?" alt="logo" width="200">
  </a>
</p>

<p align="center">a self-hostable anonymous q&a platform. visitors ask questions, you answer publicly. like ngl.link but you own the data.</p>

<p align="center">
  <a href="LICENSE"><img alt="license" src="https://img.shields.io/badge/license-O'Saasy-pink.svg" /></a>
  <a href="https://nextjs.org/"><img alt="next.js" src="https://img.shields.io/badge/next.js-15-black.svg" /></a>
  <a href="https://www.typescriptlang.org/"><img alt="typescript" src="https://img.shields.io/badge/typescript-strict-blue.svg" /></a>
  <a href="https://tailwindcss.com/"><img alt="tailwind" src="https://img.shields.io/badge/tailwind-4-38bdf8.svg" /></a>
</p>

---

### quickstart

```bash
# clone the repo
git clone https://github.com/microck/anonq.git
cd anonq

# install dependencies
npm install

# copy and configure environment
cp .env.example .env
# edit .env with your keys (see configuration section)

# run it
npm run dev
```

open `http://localhost:3000` and start asking questions.

---

### table of contents

*   [features](#features)
*   [how it works](#how-it-works)
*   [installation](#installation)
*   [configuration](#configuration)
*   [deployment](#deployment)
*   [usage](#usage)
*   [troubleshooting](#troubleshooting)

---

### features

anonq is designed to be simple, private, and self-hosted.

*   **truly anonymous:** no ip logging, no browser fingerprinting. your visitors stay anonymous.
*   **ai text rewriting:** optional llm-powered feature to disguise writing style so the sender can't be identified by how they type.
*   **admin dashboard:** secure auth0-protected panel to view, answer, and delete questions.
*   **public feed:** answered questions appear on the homepage in real-time with auto-refresh.
*   **push notifications:** get notified instantly via ntfy when someone asks a question.
*   **rate limiting:** built-in protection against spam (5 questions/hour/ip, 10 rewrites/15min).
*   **byok support:** bring your own api key for openai or any openai-compatible endpoint.
*   **supabase backend:** simple postgres database with row-level security.
*   **webgl background:** fancy terminal-style animated background (toggleable).

---

### how it works

the flow is straightforward:

1.  **visitor submits question:** anonymous form, no account required.
2.  **optional ai rewrite:** visitor can click "rewrite" to disguise their writing style before sending.
3.  **admin gets notified:** push notification via ntfy (if configured).
4.  **admin answers:** login to dashboard, read question, type answer, publish.
5.  **public sees it:** answered q&a pairs appear on the homepage feed.

```
visitor                    anonq                      admin
   |                         |                          |
   |-- submit question ----->|                          |
   |                         |-- store in supabase ---->|
   |                         |-- push notification ---->|
   |                         |                          |
   |                         |<---- answer question ----|
   |                         |                          |
   |<-- view public feed ----|                          |
```

---

### installation

#### prerequisites

*   node.js 18+
*   a supabase project (free tier works)
*   an auth0 application (free tier works)
*   (optional) openai api key for text rewriting
*   (optional) ntfy topic for push notifications

#### 1. clone and install

```bash
git clone https://github.com/microck/anonq.git
cd anonq
npm install
```

#### 2. supabase setup

create a new supabase project and run this sql in the sql editor:

```sql
-- questions table
create table questions (
  id uuid default gen_random_uuid() primary key,
  content text not null,
  timestamp timestamptz default now(),
  answered boolean default false
);

-- answers table
create table answers (
  id uuid default gen_random_uuid() primary key,
  question_id uuid references questions(id) on delete cascade,
  content text not null,
  timestamp timestamptz default now()
);

-- enable row level security
alter table questions enable row level security;
alter table answers enable row level security;

-- public can read answered questions and their answers
create policy "public read answered" on questions for select using (answered = true);
create policy "public read answers" on answers for select using (true);

-- public can insert questions
create policy "public insert questions" on questions for insert with check (true);
```

#### 3. auth0 setup

create an auth0 application (regular web app) and configure:

| field | value |
| :--- | :--- |
| allowed callback urls | `http://localhost:3000/api/auth/callback` |
| allowed logout urls | `http://localhost:3000` |
| allowed web origins | `http://localhost:3000` |

for production, add your domain equivalents.

#### 4. environment variables

copy `.env.example` to `.env` and fill in your values.

---

### configuration

| variable | required | description |
| :--- | :--- | :--- |
| `NEXT_PUBLIC_SUPABASE_URL` | yes | your supabase project url |
| `NEXT_PUBLIC_SUPABASE_ANON_KEY` | yes | supabase anon/public key |
| `AUTH0_SECRET` | yes | random string (`openssl rand -hex 32`) |
| `AUTH0_DOMAIN` | yes | your auth0 tenant domain |
| `AUTH0_CLIENT_ID` | yes | auth0 app client id |
| `AUTH0_CLIENT_SECRET` | yes | auth0 app client secret |
| `APP_BASE_URL` | yes | your app url (e.g., `http://localhost:3000`) |
| `ALLOWED_ADMIN_EMAILS` | no | comma-separated admin emails |
| `API_PROVIDER` | no | `openai` or `custom` |
| `OPENAI_API_KEY` | no | for ai text rewriting |
| `OPENAI_MODEL` | no | model to use (default: `gpt-3.5-turbo`) |
| `CUSTOM_API_URL` | no | custom openai-compatible endpoint |
| `CUSTOM_API_KEY` | no | custom api bearer token |
| `NTFY_URL` | no | ntfy topic url for push notifications |

---

### deployment

anonq can be deployed anywhere that runs node.js.

#### netlify

already configured. just connect your repo and set environment variables.

```toml
# netlify.toml is included
[build]
  command = "npm run build"
  publish = ".next"
```

#### vercel

```bash
npm i -g vercel
vercel
```

#### docker

```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
RUN npm run build
EXPOSE 3000
CMD ["npm", "start"]
```

#### self-hosted

```bash
npm run build
npm start
```

---

### usage

#### asking questions (visitor)

1. go to the homepage
2. type your question in the box
3. (optional) click "rewrite" to disguise your writing style
4. click "send question"

#### answering questions (admin)

1. go to `/admin`
2. login with auth0
3. click on a question to select it
4. type your answer
5. click "post answer"

the q&a will appear on the public feed immediately.

---

### troubleshooting

| problem | likely cause | fix |
| :--- | :--- | :--- |
| **can't login to admin** | auth0 misconfigured | check callback urls match your domain exactly |
| **questions not saving** | supabase rls | make sure the insert policy exists |
| **rewrite not working** | no api key | set `OPENAI_API_KEY` or `CUSTOM_API_KEY` |
| **no push notifications** | ntfy not set | add `NTFY_URL` to your environment |
| **rate limited** | spam protection | wait an hour (5 questions/hour limit) |

---

### license

o'saasy license
