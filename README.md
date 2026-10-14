# PingFederate OAuth 2.0 Authorization Code + PKCE load test (JMeter)

`pingfederate_pkce_loadtest.jmx` drives the full token lifecycle against PingFederate using the **authentication API (pi.flow)**, with a different user from `users.csv` on every iteration and `client_secret_basic` client authentication.

> Status: the plan is validated as well-formed XML and the PKCE algorithm is checked against the RFC 7636 test vector. It has **not been executed in JMeter** (no JMeter available where it was built). Run a 1-thread smoke test first (below) and expect to adjust assertions if your authentication policy differs from the assumptions in "PingFederate prerequisites".

## Scenario (one iteration = one user)

| Label | Endpoint | Purpose |
|---|---|---|
| `01_Authorize_pi.flow` | `GET /as/authorization.oauth2` (`response_mode=pi.flow`, S256 challenge) | Starts the flow, returns flow ID and `USERNAME_PASSWORD_REQUIRED` |
| `02_Submit_Credentials` | `POST /pf-ws/authn/flows/{id}` (`application/vnd.pingidentity.checkUsernamePassword+json`) | Username/password; expects `RESUME` + `resumeUrl` |
| `03_Resume_Get_AuthCode` | `GET {resumeUrl path}` | Returns the authorization code as JSON |
| `04_Token_Exchange_AuthCode` | `POST /as/token.oauth2` (`authorization_code` + `code_verifier`) | Access, refresh and ID tokens |
| `05a_Introspect_AccessToken` | `POST /as/introspect.oauth2` | `active=true` (works for reference and JWT tokens) |
| `05b_UserInfo` | `GET /idp/userinfo.openid` | Bearer validation, `sub` present |
| `05c_JWT_Local_Check` | client-side | exp, optional iss, RSA signature vs cached JWKS (skipped for reference tokens) |
| `06a_Token_Refresh` / `06b_Introspect_Refreshed_AT` | `POST /as/token.oauth2` (`refresh_token`) | Refresh and validate the new token; refresh-token rotation handled |
| `07a_Revoke_AccessToken` / `07b_Revoke_RefreshToken` / `07c_Introspect_After_Revoke` | `POST /as/revoke_token.oauth2` | Revoke and verify `active=false` |
| `08a_Logout_RP_Initiated` / `08b_Verify_Session_Terminated` | `GET /idp/init_logout.openid` | Logout with `id_token_hint`; a new authorization request must require login again |

Transaction rows (children stay visible as individual samples, think time excluded):

- `TX_E2E_Login_to_Tokens`: end-to-end authentication latency (authorize through token exchange)
- `TX_AuthCode_Acquisition`, `TX_Validate_Access_Token`, `TX_Refresh`, `TX_Revoke`, `TX_Logout`

"Token exchange" here means the code exchange and the refresh exchange. The RFC 8693 token-exchange grant is not included.

Endpoint paths are read at start-up from `/.well-known/openid-configuration` (setUp 2) and fall back to the documented defaults. `ep.*` properties override both. The `/pf-ws/authn/flows/` path and the media type are from the authentication API documentation.

Failures do not abort an iteration blindly: each later stage is guarded on the variables it needs, so a failed refresh still revokes and logs out with the tokens you hold.

## PingFederate prerequisites

1. Authentication API enabled for redirectless flows (Authentication > Integration > Authentication API Applications) and the option allowing OAuth clients to initiate via the authentication API. Option names vary slightly by version; check yours.
2. OAuth client: Authorization Code and Refresh Token grants, PKCE, `client_secret_basic`, your `redirect_uri` registered, `openid` scope allowed (needed for `id_token`, userinfo and logout).
3. Authentication policy for the test users resolves to the username/password adapter and then straight to `RESUME`. MFA, consent or extra steps return other `status` values and fail `02`; extend the plan for those.
4. Introspection: the client used (`client_id`, or `introspect_client_id`) must be allowed to call `/as/introspect.oauth2`.
5. JWT access tokens cannot be revoked directly. `07a` is skipped automatically for JWTs; `07c` only passes if revocation is enabled on the access token manager. Set `verify_revocation=false` otherwise.
6. RP-initiated logout needs `id_token_hint` (sent). If the PingFederate cookie is not configured, PingFederate may prompt for confirmation (HTTP 200 HTML).

## Files

- `pingfederate_pkce_loadtest.jmx`: test plan
- `env.properties`: target, client, load profile, toggles (no secrets)
- `secrets.properties.example`: copy to `secrets.properties` (`chmod 600`, git-ignored) and set `client_secret`
- `users.csv`: `username,password` with a header row; **replace the sample users**

## Users CSV

- One row is consumed per iteration, shared across all threads, so concurrent threads get different users.
- Keep at least as many rows as `threads`. The preflight sampler warns when rows < threads. When the file is exhausted it recycles, so long runs reuse users sequentially.
- Quoted fields are supported (passwords containing commas).
- Do not commit real credentials. Keep real files outside the repo (`users_file=/secure/path/users.csv`).

## Run

Requires JMeter 5.6+ and Java 17+.

```bash
# Smoke test: one user, one iteration
jmeter -n -t pingfederate_pkce_loadtest.jmx -q env.properties -q secrets.properties \
  -Jthreads=1 -Jloops=1 -l results/smoke.jtl

# Load run with HTML dashboard
jmeter -n -t pingfederate_pkce_loadtest.jmx -q env.properties -q secrets.properties \
  -Jthreads=100 -Jrampup=300 -Jduration=1800 -Jthink_ms=2000 -Jthink_range_ms=1000 \
  -l results/run1.jtl -e -o results/run1-report
```

`results/` must not already contain `run1-report` or `run1.jtl`. A non-default `users_file`, `host` or any other setting can be passed with `-J`.

CI: write `secrets.properties` from a secret store at job start (`printf 'client_secret=%s\n' "$PF_CLIENT_SECRET" > secrets.properties; chmod 600 secrets.properties`). Avoid `-Jclient_secret=...`, which is visible in the process list.

If PingFederate uses a private CA, add it to the Java truststore used by JMeter (for example `JVM_ARGS="-Djavax.net.ssl.trustStore=/path/truststore.jks -Djavax.net.ssl.trustStorePassword=..."`). Do not disable certificate validation.

### Load parameters

| Property | Default | Meaning |
|---|---|---|
| `threads` | 10 | Concurrent virtual users |
| `rampup` | 60 | Seconds to start all threads |
| `duration` | 300 | Run length in seconds |
| `loops` | -1 | Iterations per thread (-1 = until `duration`) |
| `think_ms` / `think_range_ms` | 1000 / 500 | Pause = think_ms + random(0..range) before steps 02, 05a, 06a, 07b and 08a |

## Reading results

- Dashboard: Statistics table (per label: samples, error %, mean, p90/p95/p99, throughput), Response Times Over Time, Errors, Throughput.
- End-to-end authentication latency: the `TX_E2E_Login_to_Tokens` row. Individual endpoints: the numbered rows.
- Success rate: error % per row. A step that is skipped because an earlier one failed produces no sample.
- Exclude the `Preflight`, `S1_OIDC_Discovery` and `S2_JWKS` setup rows, and `05c_JWT_Local_Check` (client-side), when quoting server throughput.
- Failure detail is in the JTL `failureMessage` column (for example which JSON assertion failed and the actual `status`).

## Security notes

- Credentials come from CSV and properties files, never from the plan. Do not enable `jmeter.save.saveservice.samplerData` or `requestHeaders`, which would write the password and Basic header into the JTL.
- Every iteration uses a fresh PKCE verifier, `state` and `nonce` (`SecureRandom`), and clears cookies so each iteration behaves as a new browser session.
- Run against a non-production environment with dedicated test users.

## Known limits

- Local JWT signature verification supports RSA (RS256/384/512). Other algorithms get claim checks only and the sample message says so.
- The `03` assertion expects a JSON body containing `code`. If the resume call returns a redirect instead, `response_mode=pi.flow` is not being honored; check the authentication API prerequisites.
