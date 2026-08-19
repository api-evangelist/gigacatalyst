# Gigacatalyst Setup Guide

Gigacatalyst is an API-connected app builder (similar to Retool or Postman). This guide finds the APIs in this project, then creates a workspace built around them.

Only API structure is used: route paths, HTTP methods, parameter names, and response shapes. No source code, credentials, or secrets are sent.

**Never open `.env`, `.env.local`, or any other secrets or config file at any point in this guide.** Route definitions and schemas already describe the API, and no credential is needed here: the human enters theirs later, inside the product, in their own signed-in session.

## Step 1 — Find the project's APIs

Look through the codebase for:

- Route definitions (REST, GraphQL, tRPC)
- OpenAPI / Swagger spec files
- API client modules
- Database schemas that back API responses
- Supabase: with `@supabase/supabase-js` present, each table is a REST endpoint via PostgREST

Note the API's base URL if checked-in source states it plainly (an OpenAPI `servers` entry, a base-URL constant, a documented public endpoint). That is an address, not a secret.

## Step 2 — Create the workspace

Continue with the setup steps at:

https://v2.gigacatalyst.com/agent.md

That page defines exactly what to send, and creates the account and saves the API description in a single request, so the human lands signed in with their API already catalogued.
