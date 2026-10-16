# PingFederate Auth Code load test (JMeter)

Replays the client's Postman collection `IAM-GKE-Load Test` under load, one user from `users.csv` per iteration.

| Step | Request | Captured |
|---|---|---|
| `1_Authorize` | `GET /as/authorization.oauth2?client_id&response_type=code&scope=openid&response_mode=pi.flow` | `flow_id` |
| `2_Check_Username_Password` | `POST /pf-ws/authn/flows/{flow_id}?action=checkUsernamePassword` (`X-XSRF-Header`, JSON `{username,password}`) | `code` (`authorizeResponse.code`) |
| `3_Token_Exchange` | `POST /as/token.oauth2`, Basic auth, `grant_type=authorization_code&code` | `access_token` |
| `4_UserInfo` | `GET /idp/userinfo.openid`, Bearer | |

`TX_Login_to_Token` wraps steps 1 to 3 and gives end-to-end login latency (think time excluded). If any request fails, that iteration stops and the thread moves on to the next user.

Status: not yet run in JMeter. Expect to adjust an assertion after the first smoke test.

## Files

- `pingfederate_authcode_loadtest.jmx`: the plan
- `env.properties`: host, client, load profile (no secrets)
- `secrets.properties.example`: copy to `secrets.properties`, `chmod 600`, set `client_secret` (the Basic-auth password of the Postman "get Auth Token" request)
- `users.csv`: 50 users from `50_users.ldif` (`uid` 1 to 50)
- `full/`: the larger plan (PKCE, introspection, refresh, revoke, logout) for later

## Before running

- The machine running JMeter must reach `fedtesting-stg.transunion.com:443` (staging may need a VM/pod inside the VPC or VPN, plus any allow-listing).
- The 50 users must exist in the directory behind PingFederate, and the login identifier must be `uid`.
- Agree the test window and load ceiling with the client.
- Requires JMeter 5.6+ and Java 17+.

## Run

```bash
mkdir -p results

# Smoke test: one user, one iteration
jmeter -n -t pingfederate_authcode_loadtest.jmx -q env.properties -q secrets.properties \
  -Jthreads=1 -Jloops=1 -l results/smoke.jtl

# Load run with HTML dashboard
jmeter -n -t pingfederate_authcode_loadtest.jmx -q env.properties -q secrets.properties \
  -Jthreads=50 -Jrampup=300 -Jduration=1800 -l results/run1.jtl -e -o results/run1-report
```

`results/run1.jtl` and `results/run1-report` must not already exist. All four rows should appear in the smoke-test summary.

## Properties

`threads`, `rampup` (s), `duration` (s), `loops` (-1 = until duration), `think_ms` + `think_range_ms`, `users_file`, `host`, `port`, `client_id`, `scope`, `authn_content_type` (fallback `text/plain`).

## Notes

- With 50 users, up to 50 threads use distinct users at once; above that, users repeat.
- The credentials body is plain JSON, so passwords containing `"` or `\` would need escaping. The current test passwords are fine.
- Do not enable `jmeter.save.saveservice.samplerData` or `requestHeaders`; the JTL would then contain passwords and the Basic header.
- Do not commit `secrets.properties`. The Postman collection contains plaintext secrets; ask the client to rotate them after testing.