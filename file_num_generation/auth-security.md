# file_num_generation — Auth and security

| Concern | How it works | Where |
|---|---|---|
| Authentication | Done by `trexo-apigateway` (`checkAuthentication`, JWT). The gateway passes the signed-in user to the service as `X-User` (JSON) and the tenant as `X-Tenant-Name` | `trexo-apigateway: routes/tenantAPI.router.js`; `src/middleware/requestContext.js` |
| Header checks | `X-User` must be present and a JSON object, and `X-Tenant-Name` must be present; otherwise a Boom 400 | `requestContext.js` |
| Identity used | `X-User.email` is stored as `generatedBy` on generation | controller |
| Module access | The frontend shows the module to the tenant named in `REACT_APP_NUMBER_GENERATOR_TENANT`, and the Admin page (user management and Export) to users whose `TENANT_ROLE` is `admin` | `trexo-frontend: pages/fileNumberGenerator/` |
| Login rules | The gateway applies an IP allow-list at login, and sends a notification on login attempts by unknown emails, for the tenant named in `NUMBER_GENERATOR_TENANT` | `trexo-apigateway: controllers/tenants/user/user.controller.js` |
| Tenancy | One numbering space per deployment: a single database (`DB_NAME`) with no tenant column. `X-Tenant-Name` appears in development log prefixes | models, `requestContext.js` |
| Input validation | Joi on every route that reads input. Unknown body keys are stripped. Validation errors are logged as field names only, not the request body | README → Request Validation; `.claude/docs/DECISIONS.md` |
| Database access | All queries go through Sequelize with bound parameters | `src/` |
| CORS | The platform domain and its subdomains, plus requests with no `Origin`; open in development | `src/index.js` |
| Secrets | DB credentials come from environment variables or AWS Secrets Manager (`USE_SECRET_MANAGER`, `DB_SECRET_NAME`). CloudWatch credentials come from environment variables. `config/config.json` is legacy and unused; `.claude/docs/CONSTRAINTS.md` says not to open it | README → Database Credentials |
| Transport | DB connections use SSL in `qa`, `uat` and `production`. `DB_SSL_REJECT_UNAUTHORIZED=true` turns on certificate verification | `src/models/index.js` |
| Container | Runs as a non-root `nodejs` user, with production dependencies only | `Dockerfile` |
| Audit trail | Each number records `generatedBy`, `createdAt` and `reason`. Save and delete actions are written to the service logs | models, controller |

When changing anything that touches the `X-User` / `X-Tenant-Name` contract, tenancy or permissions, agree it first with the gateway owner (`.claude/docs/CONSTRAINTS.md`).
