# Complete gateway authentication

This package now supplies user login, cookie sessions, JWT issuance and bearer validation. Use `Integration/CompleteAuthentication.cs.snippet` instead of the earlier TokenGeneration snippet and issuer placeholders.

1. Copy `Authentication/`, `Views/`, `wwwroot/`, controllers and services into the API. Keep `Tools/` and `Tests/` outside the API compilation: the tool is a separate console project.
2. Add `Microsoft.AspNetCore.Authentication.JwtBearer` 9.0.x. Merge the CompleteAuthentication snippet into Program.cs; call AddGatewayAuthentication AFTER AddSqlGateway, registering each only once.
3. Run `dotnet run --project Tools/ProvisionGatewayUser` in this package. Enter username and password. The tool outputs configuration containing a random signing key, stable user ID and salted ASP.NET Identity PBKDF2 password hash. Supply this through protected configuration, user secrets or a deployment secret store. Do not commit the signing key or passwords. `GatewayAuth__SigningKeyBase64` can supply the key through an environment variable.
4. For more users, run the tool again and copy ONLY the new user entry into GatewayAuth:Users. Preserve the original signing key and existing user IDs. All API instances must share the key and user configuration. Configuration/account changes require restart.
5. Open `/gateway/login` over HTTPS. Sign in, generate a token at `/gateway/token`, and download token.txt. Place it at `/jcr/Ajith_Kumar/Token/token.txt` on SAS, or supply a user-specific tokenfile= path to all macros. Each user must use their own token file.
6. SAS sends Authorization: Bearer on OPEN, EXECUTE and CLOSE. The API checks signature, HS256 algorithm, issuer, audience, expiry and enabled user status. Missing, expired or invalid tokens receive HTTP 401.

Browser cookies cannot authenticate the SQL API; it requires the GatewayApi bearer policy. The token page uses a separate HTTPS-only HttpOnly cookie, a 20-minute sign-in session and antiforgery on login, generation and logout. Login is limited to 10 attempts per minute across each application instance.

Tokens expire after 60 minutes by default. There is no automatic renewal or individual-token revocation store. Regenerate after expiry; the stable user ID lets a new token access an unexpired logical session. Sign out does not revoke previously issued JWTs. Disable a user and reload configuration to reject all their tokens, or rotate the signing key to reject all tokens.

Accounts are provisioned administratively. There is no public registration or user-selected identity in token requests. This configured account store is intended for a small managed user population. Oracle connections still use the existing WorkbenchContext.
