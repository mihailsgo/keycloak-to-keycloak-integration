# Practical test: latest Keycloak federating Smart-ID via Trustlynx

This stack runs a vanilla Keycloak locally and configures it to delegate Smart-ID authentication to the live Trustlynx Keycloak at `https://esign.trustlynx.com/ext-portal-keycloak/auth/realms/trustlynx`.

```
User browser ──▶ Local Keycloak (this repo) ──OIDC──▶ Trustlynx Keycloak (signbox) ──▶ Smart-ID
```

## What's already done on the Trustlynx side

A confidential OIDC client `local-kc-test` has been created in the `trustlynx` realm with:

- Redirect URI: `http://localhost:8081/realms/test/broker/trustlynx-smartid/endpoint`
- Web origin: `+`
- Realm-default browser flow `Authenticator` (Smart-ID-aware), no per-client override needed
- Protocol mappers for `personCode`, `country`, `firstName`, `lastName`, `email`, `authProvider`, `dateOfBirth`, `age`, `documentNumber`, `serialNumber`, `phoneNumber`, `userAccount`

The `client_secret` is in `.env` (gitignored).

## Run

```powershell
# from this folder
docker compose up -d
docker compose logs -f keycloak    # wait for "Listening on..."
```

Open: <http://localhost:8081/>

- Admin console: <http://localhost:8081/admin/> (admin / admin)
- Test realm account console: <http://localhost:8081/realms/test/account/>

## Test the Smart-ID flow

1. Open an incognito window: <http://localhost:8081/realms/test/account/>
2. The local Keycloak login page renders with the Smart-ID button:

   ![Local Keycloak login page with Sign in with Smart-ID button](docs/screenshots/11-local-login-page.png)

3. Click **Sign in with Smart-ID**. Browser is redirected to `esign.trustlynx.com`. The Trustlynx login form appears; pick the LV tab:

   ![Trustlynx LV tab with Smart-ID button](docs/screenshots/13-trustlynx-lv-tab.png)

   Click **Smart-ID**. A modal opens with a QR code and a Personal ID-code field:

   ![Smart-ID modal with QR code and Personal ID-code field](docs/screenshots/14-trustlynx-smartid-modal.png)

4. Enter your personal code, confirm on phone.
5. Browser redirects back to the local Keycloak.
6. The local Keycloak provisions a new user.
7. Verify in admin console (<http://localhost:8081/admin/>): realm `test` > Users > open the new user > Attributes tab. The fields `personCode`, `country`, `firstName`, `lastName`, `email`, `authProvider` should be populated.

## Stop / clean up

```powershell
docker compose down            # keeps DB
docker compose down -v         # also wipes the realm DB so next "up" re-imports
```

## When done with testing, clean up trustlynx side

The `local-kc-test` OIDC client on the trustlynx server is harmless but not free of cleanup. To remove it:

- Admin console: `https://esign.trustlynx.com/ext-portal-keycloak/auth/admin/` > realm `trustlynx` > Clients > `local-kc-test` > Delete.
- Or via API:
  ```bash
  TOKEN=$(curl -s -X POST "https://esign.trustlynx.com/ext-portal-keycloak/auth/realms/master/protocol/openid-connect/token" \
    -d "grant_type=password" -d "client_id=admin-cli" -d "username=admin" -d "password=<admin-pw>" \
    | python -c "import sys,json;print(json.load(sys.stdin)['access_token'])")
  CID=$(curl -s -H "Authorization: Bearer $TOKEN" \
    "https://esign.trustlynx.com/ext-portal-keycloak/auth/admin/realms/trustlynx/clients?clientId=local-kc-test" \
    | python -c "import sys,json;print(json.load(sys.stdin)[0]['id'])")
  curl -X DELETE -H "Authorization: Bearer $TOKEN" \
    "https://esign.trustlynx.com/ext-portal-keycloak/auth/admin/realms/trustlynx/clients/$CID"
  ```

## Files

- [docker-compose.yml](docker-compose.yml): Postgres + Keycloak 26.5.1
- [realm/test-realm.json](realm/test-realm.json): local `test` realm. Contains the IdP, attribute mappers, duplicated `first broker login (Smart-ID)` flow, and declarative user profile with `unmanagedAttributePolicy: ENABLED`
- [.env.example](.env.example): template for `.env`
- [README.md](README.md): customer-facing integration guide (the production procedure this stack mirrors)

## Verified end-to-end

A live Smart-ID login was driven through this stack with a real LV personal code and phone confirmation. After the flow, the resulting user in the local `test` realm carried:

| Field | Origin |
| --- | --- |
| username | a 64-char hex string (`sub` claim from trustlynx, do **not** rely on this; see [README.md](README.md) §7 "Stable identity guidance") |
| firstName | Trustlynx claim, mapped |
| lastName | Trustlynx claim, mapped |
| email | typed once on Update Account form (Smart-ID does not return email) |
| `personCode` | Trustlynx claim, mapped |
| `country` | Trustlynx claim, mapped |
| `authProvider` | Trustlynx claim, mapped (value `DM_SMART_ID`) |

The stable identity is `country + ":" + personCode`; see [README.md](README.md) §7 "Stable identity guidance".

## Gotchas discovered while wiring this up

1. **Trustlynx requires PKCE** on its OIDC client (because the protocol mappers we copied from the existing `trustlynx-signing-portal` client carry `pkce.code.challenge.method: S256` as a hard requirement). The local IdP must therefore be configured with `pkceEnabled: true` and `pkceMethod: S256`. Without that you get `invalid_request: Missing parameter: code_challenge_method` on the trustlynx side.

2. **Keycloak 26 declarative user profile drops unknown attributes** by default. Custom attributes like `personCode`, `country`, `authProvider` will silently disappear unless you either (a) declare them in the user profile config, or (b) set `unmanagedAttributePolicy: ENABLED` on the realm. We do (b) in [realm/test-realm.json](realm/test-realm.json), which is the simplest option and matches what most IdP integrations want.

3. **Trustlynx Smart-ID does not return `email`**. Either let Keycloak prompt the user for email on first login (the default; what this test does), or remove the email-required validation from the realm's user profile if you want fully silent provisioning.

4. **`conditional-otp-form` authenticator no longer exists in KC 26.** It was the canonical example in older Keycloak docs for adding 2FA inside a sub-flow. In modern KC the conditional 2FA is a separate sub-flow with `conditional-credential` + `auth-otp-form`. We dropped the OTP path from the duplicated first-broker-login flow entirely. For an IdP that already provides strong auth (Smart-ID), an extra OTP step is redundant.
