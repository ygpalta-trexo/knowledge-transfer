# file_num_generation — Deployment and environments

## Environments

| `NODE_ENV` | Meaning in code |
|---|---|
| `development` | Loads `.env*`, console plus file logs, CORS open, no DB SSL |
| `test` | No `.env` loading, silent logs, local database only (host guard), Secrets Manager forced off |
| `qa`, `uat`, `production` | DB SSL on; CloudWatch log group `<NODE_ENV>/server`; matching `sequelize-cli` config blocks with SSL |

Use one of these `NODE_ENV` values for a deployed environment: DB SSL and the `sequelize-cli` config blocks are keyed on them.

Git branches: `main`, `dev`, `qa`, `staging`, `uat`, `release`.

## Build

`Dockerfile`:
1. Start from `node:24-alpine` and add a non-root `nodejs` user.
2. `npm ci --omit=dev`, then copy the repo.
3. Set `ENV NODE_ENV=production` and a `PORT`, and `EXPOSE` that port.
4. `CMD npm start`.

The code defaults to port 8015 when `PORT` is unset. Set `NODE_ENV` and `PORT` per environment in the runtime configuration, and point the gateway's `FILE_NUMBER_GENERATOR_API_URL` at that port.

## Database changes

Migrations and seeding are manual `sequelize-cli` steps; the service does not run them at start-up. Run them with the target environment's variables, including `USE_SECRET_MANAGER` / `DB_SECRET_NAME` when used:

```
NODE_ENV=<env> npx sequelize-cli db:migrate:status
NODE_ENV=<env> npx sequelize-cli db:migrate
```

Run `db:seed:all` only on a new database, once: it inserts the 9 counter rows, and `(matterType, subOrg)` is unique. For a new database, see [database.md](database.md#setting-up-a-new-database).

## Tests in a pipeline

Run `npm run test:coverage` against a throwaway Postgres service, on Node 18 or later (Jest 30 and Express 5 require it). The coverage gate is 75 % on all four metrics (`jest.config.js`). `.claude/docs/TESTS.md` has the expected output.

## Configuration (variable names only)

| Variable | Used for | Default |
|---|---|---|
| `NODE_ENV` | Environment switch (see above) | `production` in the image |
| `PORT` | HTTP port | 8015 in code |
| `USE_SECRET_MANAGER` | `"true"` → DB credentials from Secrets Manager only | unset → env vars |
| `DB_SECRET_NAME` | Secret id (JSON `username`, `password`, `host`, `engine`) | — |
| `DB_CREDENTIALS_TTL_MS` | Credential cache TTL | 600000 |
| `AWS_REGION` | Secrets Manager and CloudWatch region | `us-east-1` for Secrets Manager |
| `DB_NAME` | Database name (required in both modes) | — |
| `DB_HOST`, `DB_USERNAME` (or `DB_USER`), `DB_PASSWORD` | Env-mode credentials | — |
| `DB_DIALECT` | Sequelize dialect | `postgres` |
| `DB_SSL_REJECT_UNAUTHORIZED` | `"true"` → verify the RDS certificate | off |
| `FILE_NUMBER_STARTING_COUNTER` | JSON `{SUBORG: {MATTER_TYPE: "Y1Y2NNN"}}`: the position a counter is set to when its series has been emptied by deletes | Built-in values ([product-and-features.md](product-and-features.md#business-rules)) |
| `LOG_LEVEL` | Winston level | Winston default |
| `LOG_RETENTION_DURATION_IN_DAYS` | CloudWatch retention | — |
| `AWS_ACCESS_KEY_ID`, `AWS_ACCESS_KEY_SECRET` | CloudWatch transport credentials | — |

Gateway side: `FILE_NUMBER_GENERATOR_API_URL`, `NUMBER_GENERATOR_TENANT`, `LOCAL_TENANT`. Frontend side: `REACT_APP_NUMBER_GENERATOR_TENANT`.

## Infrastructure dependencies

- Postgres.
- AWS Secrets Manager (when `USE_SECRET_MANAGER=true`).
- CloudWatch Logs and Metrics.
- `trexo-apigateway` reaching the service at `FILE_NUMBER_GENERATOR_API_URL`.
- A writable `logs/` directory in the container, for development logs and undelivered CloudWatch batches. The `Dockerfile` gives the `nodejs` user ownership of the app directory.
