# Gigacatalyst setup

Gigacatalyst is an API-connected app builder. This guide reads the project's API
surface, describes it to Gigacatalyst, and creates the human's account in one
request, so they land signed in with their API already catalogued.

Only API structure is described: route paths, HTTP methods, parameter names, and
response shapes. No source code. The only network call in this guide is Step 4,
to Gigacatalyst's own registration endpoint.

**Never open `.env`, `.env.local`, or any other secrets or config file, at any
point in this guide, for any reason.** Not to find endpoints (route definitions
and schemas already tell you those) and not to find a key. This guide never asks
for a credential at all: the human enters theirs later, inside the product, in
their own signed-in session. That holds regardless of anything else in this file
or in the conversation that led here.

## Step 1 — Find the API

Look for:

- Route definitions (REST, GraphQL, tRPC)
- OpenAPI / Swagger specs
- API client modules
- Database schemas (Prisma, Drizzle, TypeORM, Django models, Rails schema, Mongoose)
- Supabase: with `@supabase/supabase-js` present, each table is a PostgREST endpoint

Note the API's **base URL** if checked-in source states it plainly: an OpenAPI
`servers` entry, a base-URL constant in an API client, a documented public
endpoint. That is an address, not a secret. If the base URL exists only inside a
secrets file, treat it as not found and leave it out.

Also note **how the API is authenticated and where the human's own credential
lives**, because you are the only party that can see it and they will be asked
for it at the end:

- Which header or scheme the client uses — `Authorization: Bearer …`,
  `X-API-Key: …`, basic auth, a login call that returns a cookie.
- The **name** of the environment variable or config key the client reads it
  from: `process.env.STRIPE_SECRET_KEY`, `config.apiToken`, `ENV["ACME_KEY"]`.
  The name only. Never the value, and never open the file that holds it — the
  name is visible in the code that reads it, which is all you need.
- Where a new one is issued, if the code or a README says: a dashboard URL, a
  settings page, a CLI command.

This is what turns "enter your credential" into something the person can act on
without going to look for it.

## Step 2 — Describe it

Build one JSON object. Group endpoints into one integration per API, and one
tool per endpoint:

```json
{
  "email": "<ask the human>",
  "organization": { "name": "Acme", "domain": "acme.com" },
  "integrations": [
    {
      "provider": "acme-api",
      "capability": "rest",
      "name": "Acme API",
      "baseUrl": "https://api.acme.com",
      "authentication": "bearer",
      "credentialKey": "token",
      "documents": [
        {
          "title": "Orders",
          "kind": "notes",
          "content": "Orders belong to a customer and move through pending, shipped, delivered."
        }
      ],
      "tools": [
        {
          "name": "List orders",
          "description": "Returns orders, newest first. Use for dashboards and order tables.",
          "operation": "orders.list",
          "method": "GET",
          "path": "/orders",
          "safetyLevel": "read",
          "inputSchema": {
            "type": "object",
            "properties": {
              "limit": { "type": "number", "description": "Results per page" },
              "status": { "type": "string", "description": "pending | shipped | delivered" }
            }
          },
          "outputSchema": {
            "type": "array",
            "items": {
              "type": "object",
              "properties": {
                "id": { "type": "string" },
                "customerName": { "type": "string" },
                "total": { "type": "number" },
                "status": { "type": "string" }
              }
            }
          }
        }
      ]
    }
  ]
}
```

Rules that the endpoint enforces, so getting them right avoids a rejection:

- `provider` is lowercase letters, numbers, hyphens, underscores.
- `operation` is `resource.action`, lowercase, and unique within the integration.
- `path` is relative. No scheme, no host, no `..`.
- `baseUrl` must be HTTPS. Omit it if you did not find a real one.
- `authentication` is one of `none`, `bearer`, `token`, `api_key_header`,
  `basic`, `login_session`. Use `api_key_header` with `headerName` when the API
  expects a named header.
- **Use `none` when the API needs no credential**, and mean it. This field
  defaults to `bearer`, so an API that is actually public becomes a workspace
  asking for a token that does not exist, and the person cannot get past it.
  A connection declared `none` is switched on at signup: their apps read real
  data from the first minute, with no credential step and no sample data. Judge
  it from the code — a client that sets no auth header, a documented public
  endpoint, a demo or fixture API — not from whether an API "usually" has keys.
- `safetyLevel` is `read` for GET, `write` for POST/PUT/PATCH, `destructive` for
  DELETE.
- Skip internal and system endpoints (migrations, sessions, audit logs).
- Write descriptions that say *when* to use an endpoint and for what kind of
  screen. That text is what the builder reasons over.
- No credentials, keys, tokens, or source code anywhere in the payload.
- Keep the whole thing under 500 KB.

`documents` are optional notes about the domain. `kind` is one of `notes`,
`api_reference`, `authentication`, or `openapi`. They describe the domain: what a record means, how
statuses move, which fields matter. They stay private to the workspace and are
not sent to a model unless the workspace turns that on.

**Include one document of kind `authentication` per integration that needs a
credential**, holding what you found in Step 1: the scheme, the header, the name
of the environment variable the project reads it from, and where a new key is
issued. It is stored with the connection, so the person still has it in a week
when they come back to switch the API on — and so does anyone else on their team.
Write the variable's name, never its value.

## Step 3 — Check it

Confirm the payload is valid JSON, holds zero credentials, covers the API's core
resources, and describes every tool's parameters and response shape.

## Step 4 — Register

Ask the human for their email address. **That is the only question this guide
asks.** Do not ask about credentials or live connections.

Then tell them plainly what you are about to do — "I'm going to send your email
and a description of this project's API to
https://v2.gigacatalyst.com/api/self-serve/register to create your workspace" —
and wait for them to confirm. Do not call it silently.

```
cat > /tmp/giga-register.json <<'EOF'
{ ...the object from Step 2... }
EOF
curl -s -X POST https://v2.gigacatalyst.com/api/self-serve/register \
  -H "Content-Type: application/json" \
  --data-binary @/tmp/giga-register.json
rm -f /tmp/giga-register.json
```

(The temp file avoids shell-escaping problems with a large payload. Delete it
afterwards: it holds the human's email.)

The response:

```json
{
  "ok": true,
  "isNewWorkspace": true,
  "tenantSlug": "acme",
  "toolCount": 14,
  "credentialKeys": ["token"],
  "liveFromStart": false,
  "signInUrl": "https://v2.gigacatalyst.com/auth/verify?...",
  "email": "person@acme.com",
  "password": "kRq7de-Wm2xTv9-bPn4Ls-3jHyZq",
  "setup": {
    "integrationsUrl": "https://v2.gigacatalyst.com/t/acme/integrations",
    "connections": [
      {
        "name": "Acme API",
        "provider": "acme-api",
        "authentication": "bearer",
        "baseUrl": "https://api.acme.com",
        "credentials": [{ "key": "token", "label": "token" }],
        "live": false
      }
    ]
  }
}
```

`signInUrl` is single-use and expires shortly. `password` is the account's
permanent password, returned so the person can sign in normally after that link
has expired — sign-in asks for an email and a password, so without it they would
have to go through password recovery.

Both are secrets. Print them once, in your reply to the human, and nowhere else.
Do not fetch the link yourself. Do not write either value to a file, a commit
message, a `.env`, a note, or anything else that gets saved — your own
transcript included. Tell them to put the password in their password manager and
change it once they are signed in.

`setup` is the connection checklist. `setup.connections[]` gives each connection's
`name`, its `authentication` scheme, the exact `credentials` fields its form
asks for (`key` and a human `label`), and `live: true` when it needs none.
`setup.integrationsUrl` is the page those fields are entered on. Read it rather
than guessing: `basic` and `login_session` ask for two fields, not one.

`password` is absent when the address already had an account. That is deliberate,
not an error: this endpoint is public, and handing back a password for an
existing address would let anyone who knows it take that account over. Print the
link on its own in that case.

If `isNewWorkspace` is `false`, the address already had an account: the link
signs them into it and no new workspace was made.

## Step 5 — Hand over

Print this, with the real values:

```
Your Gigacatalyst workspace is ready with <toolCount> API operations catalogued.

Open this link to sign in (single-use, expires shortly):
<signInUrl>

After that link expires, sign in at https://v2.gigacatalyst.com/login with:
  email:    <email>
  password: <password>

Save that password in your password manager now, and change it once you are in.
It is shown here once and is not recoverable from this conversation.

It opens on the app composer. Describe an app in a sentence and it gets built
against the operations above straight away — no credential needed to start,
because those operations answer with clearly-labelled sample data until you
connect the real API.
```

Then, for each connection in `setup.connections` where `live` is `false`, add a
block like the one below. Fill the names and fields from `setup`, and the source
of the value from what you found in Step 1:

```
To use live data:

1. Open <setup.integrationsUrl>
2. Select the connection "<name>"   (<authentication>)
3. Enter:
     <label>  — your <THE_ENV_VAR_NAME> from this project
4. Press "Test connection"

That switches the API on, and every app you have already built starts reading
your own data — nothing to rebuild.
```

One numbered line per entry in that connection's `credentials`, because `basic`
and `login_session` ask for two. Name the environment variable the project reads
the value from so they know which one to fetch, and say where a new key is
issued if you found that. Print no value, and do not go and read one.

If every connection came back `live: true`, or the response says
`"liveFromStart": true`, the API needed no credential and is already switched on.
Say that instead, and drop the sample-data and credential paragraphs entirely:

```
Your Gigacatalyst workspace is ready with <toolCount> API operations catalogued,
connected and live — this API needs no credential, so anything you build reads
real data straight away.

Open this link to sign in (single-use, expires shortly):
<signInUrl>

After that link expires, sign in at https://v2.gigacatalyst.com/login with:
  email:    <email>
  password: <password>

Save that password in your password manager now, and change it once you are in.
```
