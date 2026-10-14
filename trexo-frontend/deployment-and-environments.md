# Deployment and environments: AIA-81A / AIA-83

## Branches

`main` is the default branch. The environment branches are `uat`, `release`, `qa` and `dev`. `release` carries the `Dockerfile` and `docker-compose.yml`.

## Build

- `npm ci`, then `npm run build`. This runs `react-scripts build` with `NODE_OPTIONS=--max_old_space_size=4096`. `prebuild` applies the two patch scripts in `scripts/`.
- `Dockerfile` (on `release`): `node:18-alpine` runs `npm install` and `npm run build`. Then `nginx:alpine` serves the build with `nginx.conf`, which falls back to `index.html` for SPA routes. Port 80.
- **`REACT_APP_*` values are baked in at build time.** Changing a JUNE limit needs a rebuild and redeploy.

## CI

`.github/workflows/dev.yml` builds (`npm ci`, `npm run build`) on pushes to `releasedev`, on a self-hosted runner with Node 18. It does not run tests or deploy.

## Configuration variables used by this feature (names only)

| Where | Variable | Use |
|---|---|---|
| trexo-frontend | `REACT_APP_API_BASE_URL` | gateway base URL |
| trexo-frontend | `REACT_APP_JUNE_APPLICATION_NUMBER_LIMIT` | maximum application numbers per request. Set it in every build |
| trexo-frontend | `REACT_APP_JUNE_CHARACTER_LIMIT` | chat text limit. The AIA forms always use `largeSlotData`, so it does not apply to them |
| trexo-apigateway | `TREXO_ASSISTANT_API_URL` | proxy target for `/tenantAPI/trexo-assistant` |
| patent-asistant-apis | `LEX_BOT_ID`, `LEX_BOT_ALIAS_ID`, `LEX_LOCALE_ID` | which Lex bot and alias the chat talks to |
| patent-asistant-apis | `EXCLUDED_INTENT_LIST` | intents that skip the permission check |
| patent-asistant-apis | `IP_ASSISTANT_BUCKET_NAME` | template bucket |
| patent-asistant-apis | `FORM_GENERATION_CONCURRENCY` | parallel application numbers per request (default 3) |
| patent-asistant-apis | `ODP_API_URL`, `ODP_API_KEY` | USPTO Open Data Portal |
| patent-asistant-apis | `AWS_REGION`, `AWS_BUCKET_NAME`, `AWS_S3_URL_EXPIRATION` | output upload and link expiry |
| patent-asistant-apis | `EMAIL_CONFIG_AUTH_USER`, `EMAIL_CONFIG_AUTH_PASSWORD` | output e-mail |
| pa_lambda_functions (LexChatbot) | `ACTION_BASE_URL` | base URL of the `/forms/:formCode` generator. Others are listed in [../pa_lambda_functions/deployment-and-environments.md](../pa_lambda_functions/deployment-and-environments.md) |

## Releasing a change to these forms

Most changes span repos. Ship them in this order so that no environment sends a value the next hop does not understand:

1. **Generator** (patent-asistant-apis): accept the new or renamed field, keep the old one, and update `form_templates.mappings` if a PDF field changes. Mappings are database rows, so apply the data change in each environment.
2. **Lambda** (pa_lambda_functions): forward the field. See that package for how the Lambda is bundled and deployed.
3. **Frontend**: send the field.
4. **Lex**: needed only if a slot or prompt changes, in the AWS console per bot alias.

A frontend-only change (copy, layout, client-side validation) ships alone. After any change, open one generated PDF and check every box and field before relying on the change.
