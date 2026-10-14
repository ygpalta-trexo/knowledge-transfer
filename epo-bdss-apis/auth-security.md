# epo-bdss-apis — Auth and security

The data is public patent bibliographic data. *Which* references a tenant looks up reveals a client's work, so treat request logs and reference lists as confidential.

## Mechanisms

| Concern | Mechanism | Where |
|---|---|---|
| Callers | `reference_downloader_apis` and `patent-asistant-apis`, server to server, forwarding the end user's `X-User` and `X-Tenant-Name` | callers |
| Header check | `/api` routes require `X-User` (JSON) and `X-Tenant-Name` | `src/index.js` |
| CORS | Not enabled; the service is called server to server | `src/index.js` |
| Swagger | `/api-docs` plus `POST /api/v1/auth/token` (credentials from `AUTH_USERNAME` / `AUTH_PASSWORD`, HS512 JWT signed with `SECRET`), development and qa only | `src/routes/swagger-auth.router.js` |
| DB access | Read-only by design (`main:.claude/docs/CONSTRAINTS.md`); queries use Sequelize `where` objects and `Op.in` | routes |
| DB credentials | Env or Secrets Manager; if Secrets Manager cannot be read, env credentials are used and a warning is logged | `src/config/dbCredentials/configResolver.js` |
| Local config | `.env` is gitignored, and `.dockerignore` excludes `.env*` | repo |
| Container | Non-root user | `Dockerfile` |

## Configuration note

Two AWS secret-key names appear in `.env.example`: `AWS_SECRET_ACCESS_KEY` (SDK default chain, used for Secrets Manager) and `AWS_ACCESS_KEY_SECRET` (the CloudWatch transport). Set both where needed.
