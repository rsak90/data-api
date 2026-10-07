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





# Linux deployment with Nginx

The scripts deploy the **published existing WorkBench API with the SQL Gateway integrated**. The source ZIP is not a runnable application: it still requires the real WorkbenchContext and Oracle configuration from your solution. No server connection has been supplied and no remote deployment has been performed.

Target: Ubuntu 24.04 LTS (or Debian with equivalent prerequisites), systemd 247 or later, x64 or ARM64. The API is published with its .NET runtime included. Nginx serves HTTPS and forwards to Kestrel bound only to 127.0.0.1:5080. A dedicated unprivileged service user runs the API.

## 1. Integrate the source and hosting configuration

Copy the API source folders into your WorkBench project as described in README.md and AUTHENTICATION.md. Do not copy Tools or Tests into the API source compilation. Add the JWT bearer dependency and use `Integration/CompleteAuthentication.cs.snippet`.

Merge **Integration/LinuxHosting.cs.snippet** too:

- Load production.json from the systemd credential directory before registering services. This provides the signing key, users and Oracle settings to the application.
- Configure forwarded headers to trust only the local Nginx proxy. Call UseForwardedHeaders before HTTPS redirection and authentication, so the token page works correctly behind TLS termination.
- Persist ASP.NET Data Protection keys to `/var/lib/workbench-gateway/keys`, preserving browser cookies across releases.

Keep the existing WorkbenchContext, CipherService and Oracle registration. The service can only write to `/var/lib/workbench-gateway` and its private temporary directory. Point existing file log sinks at the state directory or use console logging captured by systemd. If your API needs other writable paths, explicitly add them to the service unit.

## 2. Provision credentials

On a development/build machine with the .NET SDK:

```bash
dotnet run --project Tools/ProvisionGatewayUser
```

Enter a username and password. Merge the generated GatewayAuth object into a private production.json using `Deployment/production.example.json` as the shape. Add the **existing WorkBench Oracle configuration using the exact keys the context expects**; the scripts do not invent a connection string or decryption convention. Set AllowedHosts to the deployment hostname.

For more users, retain the original signing key and copy only the additional user entries. Preserve IDs between deployments. Restrict the file to the deployment operator:

```bash
chmod 600 /secure/path/production.json
```

The deployment script stores it root-only in /etc/workbench-gateway and systemd delivers a private readable credential file to the API. Account changes require restarting/redeploying. Never put real credentials in the example JSON or source control.

## 3. Build a Linux publish folder

On a build machine with Bash and .NET SDK 9 or later:

```bash
bash Deployment/publish.sh /path/to/WorkBench.Api.csproj /path/to/build-output linux-x64
```

Use linux-arm64 for ARM64 servers. The script prints the exact generated publish directory, such as `/path/to/build-output/publish.ABC12345`. Record it for the following commands. Match --app to the apphost executable in that directory; it is typically the API project's assembly name without `.dll`.

Windows alternative, using PowerShell:

```powershell
dotnet publish C:\path\WorkBench.Api.csproj -c Release -r linux-x64 --self-contained true -p:UseAppHost=true -o C:\path\publish
```

Keep the complete publish output. Razor pages should be compiled into the published app and wwwroot must be present. A self-contained deployment still requires the supported Linux native dependencies (including ICU/OpenSSL).

## 4. Prepare DNS, networking and TLS

Point your hostname (for example workbench.example.com) at the server. Allow inbound TCP 443 from browsers and SAS Viya; TCP 80 is used only to redirect to HTTPS. Allow outbound Oracle connectivity from this server. Do not open the Kestrel port publicly.

Provide a TLS certificate chain and private key on the server, using your internal PKI or existing certificate management. The certificate must match the hostname and be trusted by both browsers and SAS. The script accepts existing certificate files; certificate issuance and renewal are managed separately. Example paths:

```text
/etc/ssl/workbench/fullchain.pem
/etc/ssl/workbench/privkey.pem
```

The deployment verification requires a chain trusted by the server too. Install your internal CA in its trust store if applicable. The script does not disable certificate verification or alter firewall rules. Retain SSH access when changing firewall policy.

## 5. Transfer the published API and scripts

From the build machine (replace the sample paths and hostname):

```bash
ssh deploy@linux-server 'mkdir -p ~/workbench-deploy/publish'
scp -r /path/to/build-output/publish.ABC12345/. deploy@linux-server:~/workbench-deploy/publish/
scp Deployment/deploy.sh deploy@linux-server:~/workbench-deploy/
scp /secure/path/production.json deploy@linux-server:~/workbench-deploy/production.json
ssh deploy@linux-server 'chmod 700 ~/workbench-deploy; chmod 600 ~/workbench-deploy/production.json'
```

All scripts use LF line endings. If an editor changed them to CRLF, convert them back before running.

## 6. Run the automated deployment

SSH into the server and run:

```bash
cd ~/workbench-deploy
sudo bash deploy.sh \
  --publish "$PWD/publish" \
  --app WorkBench.Api \
  --domain workbench.example.com \
  --cert /etc/ssl/workbench/fullchain.pem \
  --key /etc/ssl/workbench/privkey.pem \
  --config "$PWD/production.json" \
  --port 5080 \
  --install-deps
```

`--install-deps` installs Nginx, curl, jq, ICU and CA certificates using apt on Debian/Ubuntu. Omit it when dependencies are already installed. The script assumes systemd and nginx.conf includes `/etc/nginx/conf.d/*.conf` in the HTTP context. It manages only one gateway installation per server using the fixed names below. It does not remove existing Nginx sites; resolve any hostname/port conflicts first.

The script automatically:

1. Validates inputs, published files, protected configuration and prerequisites.
2. Creates the service user, state directories and a new versioned release.
3. Saves backups of its existing service, Nginx configuration and production settings.
4. Installs root-only production settings and generates the systemd service and Nginx TLS site.
5. Validates Nginx, switches the current release, and starts the API.
6. Checks local startup, HTTPS login-page availability and HTTP 401 rejection for unauthenticated API calls.
7. Restores the previous release/configuration on a detected failure after configuration changes begin.

Paths managed by the script:

| Item | Path |
| --- | --- |
| Versioned app releases | /opt/workbench-gateway/releases |
| Active app | /opt/workbench-gateway/current |
| Protected settings | /etc/workbench-gateway/production.json |
| Service | /etc/systemd/system/workbench-gateway.service |
| Nginx site | /etc/nginx/conf.d/workbench-gateway.conf |
| Cookie keys and app state | /var/lib/workbench-gateway |

Updates briefly restart the API; active logical sessions are lost because they live in IMemoryCache. SAS must open a new session afterward. Retained previous releases/config backups allow recovery but consume disk and contain secrets; manage retention under the server's backup policy. Dependencies installed by apt and newly created user/directories are not removed during rollback.

## 7. Generate a token and connect SAS

Open `https://workbench.example.com/gateway/login`. Sign in, generate a token and download token.txt. Transfer it to your user's SAS-accessible path and restrict access.

```sas
%dbcon(baseurl=https://workbench.example.com,
       tokenfile=/jcr/Ajith_Kumar/Token/token.txt);
%dbsql(sql=%nrstr(SELECT 1 AS OK FROM dual), out=work.gateway_check,
       tokenfile=/jcr/Ajith_Kumar/Token/token.txt);
%dbdisconnect(tokenfile=/jcr/Ajith_Kumar/Token/token.txt);
```

The automated smoke checks do not run SQL or verify Oracle connectivity. This SAS SELECT verifies the database path separately. Tokens default to 60-minute expiry; generate a replacement after expiry.

## 8. Inspect and update

```bash
sudo systemctl status workbench-gateway --no-pager
sudo journalctl -u workbench-gateway -n 100 --no-pager
sudo journalctl -u workbench-gateway -f
sudo nginx -t
sudo systemctl status nginx --no-pager
```

Publish the next release and repeat step 6 with the new publish folder and the same signing key/user IDs. After certificate renewal, run `sudo nginx -t` then `sudo systemctl reload nginx`.

For manual recovery, select a known previous release path from `/opt/workbench-gateway/releases`, switch `/opt/workbench-gateway/current` to it with `sudo ln -sfn <verified-release-path> /opt/workbench-gateway/current`, then restart the service. Restore the matching protected configuration backup too if configuration changed. Do not delete the Data Protection key directory.

Troubleshooting: HTTPS redirect loops usually mean forwarded-header middleware is missing or in the wrong order. HTTP 503 from token generation means the real GatewayJwtIssuer registration is not active. A 502 means Kestrel did not start or the configured port differs. Startup failures commonly mean invalid JWT configuration, missing Oracle/context settings, wrong Linux CPU architecture or denied file writes. Check the service logs without printing the protected configuration.

## References and validation scope

Hosting follows [Microsoft's Nginx hosting guidance](https://learn.microsoft.com/en-us/aspnet/core/host-and-deploy/linux-nginx) and [Nginx proxy documentation](https://nginx.org/en/docs/http/ngx_http_proxy_module.html). .NET 9 reaches end of support on November 10, 2026 according to [Microsoft's support policy](https://dotnet.microsoft.com/en-us/platform/support/policy/dotnet-core); deployment remains on the requested .NET 9 target, with an upgrade needed for longer-term servicing.

These scripts are supplied for execution on your Linux server. Deployment and Oracle/TLS integration cannot be verified from this Windows workspace without server access.

