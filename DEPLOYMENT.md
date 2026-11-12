# CSCPO Awards portal: production deployment

## Why uploading the source ZIP to FTP returns 403

This is a full-stack Next.js application, not a folder of plain HTML files. Pages and API routes are served by a Node.js process and the nomination, voting, and admin workflows depend on PostgreSQL. A normal FTP/static Apache directory cannot start that server. Uploading the project or `.next` directory to `public_html` does not deploy the application; Apache may respond with 403/404 because it has no configured entry point or application process.

**Keep the full-stack app on a Node.js-capable host. FTP can transfer the deployment bundle to that host, but FTP by itself cannot run it.** A shared hosting account must offer a Node.js application feature (for example, cPanel Setup Node.js App) or a reverse proxy to a Node service. Otherwise use a Node-capable host.

The Arena preview URL is workspace-protected and may show “Preview Unavailable” or a traffic-token 403 in another browser. It is not a production/public URL and cannot be made public by changing the Next.js app. A public URL exists only after the Node app is deployed to a domain CSCPO controls, such as `awards.cscpo.org`.

## Recommended CSCPO setup

1. Deploy this application to a Node.js host or a cPanel Node.js application. A dedicated subdomain such as `awards.cscpo.org` is recommended.
2. Provision a PostgreSQL database that is reachable from that host. Set `DATABASE_URL` in the host's server-side environment; do not copy the development `.env` file into a public web directory.
3. Use Node.js 20 or newer, install dependencies, and build the app:

   ```sh
   npm ci
   npm run build
   ```

4. Prepare the standalone server files:

   ```sh
   bash scripts/prepare-standalone-deploy.sh
   ```

   The prepared application is in `.next/standalone/`. Transfer that folder to the Node application's application root (FTP/SFTP may be used as a file-transfer method when the host supports Node.js). The `public` and `.next/static` assets are copied into the standalone folder by the script.

5. Configure the Node host:
   - Application root: the uploaded `.next/standalone` folder
   - Startup file: `server.js`
   - Environment: `NODE_ENV=production`, plus the provider-assigned `PORT` and production `DATABASE_URL`
   - Set a private `VOTER_SESSION_SECRET` (generate one with `openssl rand -hex 32`)
   - Set `PUBLIC_APP_URL=https://awards.cscpo.org` (use the exact deployed origin)
   - Start/restart the Node application from the host's Node.js control panel
6. Configure the authentication providers in the Node host's private environment:
   - **Email OTP**: `RESEND_API_KEY` and `RESEND_FROM_EMAIL` (the sender domain must be verified in Resend).
   - **Google**: `GOOGLE_CLIENT_ID` and `GOOGLE_CLIENT_SECRET`. Register `https://awards.cscpo.org/api/auth/oauth/google/callback` as an authorized redirect URI in Google Cloud Console.
   - **Facebook**: `FACEBOOK_APP_ID` and `FACEBOOK_APP_SECRET`. Register `https://awards.cscpo.org/api/auth/oauth/facebook/callback` in the Meta app's Valid OAuth Redirect URIs. Optionally set `FACEBOOK_GRAPH_VERSION` (defaults to `v27.0`).
   - Never place provider secrets in client-side code or a public FTP directory. If these values are missing, the corresponding sign-in action returns a clear configuration error rather than creating a fake voter.
7. Confirm `https://awards.cscpo.org/api/health` returns `{"ok":true,...}` before embedding it.
8. In the WordPress/HTML page at `https://cscpo.org/cscpobdawards2026/`, embed the public gallery using the **actual deployed app origin**:

   ```html
   <iframe
     src="https://awards.cscpo.org/voting?embed=true"
     width="100%"
     height="950"
     frameborder="0"
     loading="lazy"
     title="CSCPO South Asian Leadership Awards - Public Voting"
     style="border:none; border-radius:16px; overflow:hidden;">
   </iframe>
   ```

   Replace `awards.cscpo.org` if CSCPO chooses a different Node-hosted domain. The WordPress event page and the voting app are separate deployments; uploading app files into the WordPress FTP directory does not make its API routes run.

## Direct routes

- Home: `/`
- Voting gallery: `/voting`
- Nomination portal: `/nominate`
- Results: `/results`
- Admin sign-in: `/admin`
- Access links: `/access`
- Health check: `/api/health`

## Local / self-hosted standalone start

After `npm run build` and `bash scripts/prepare-standalone-deploy.sh`, run the prepared bundle with the production environment configured:

```sh
cd .next/standalone
node server.js
```

The process must remain running under the host's process manager. Keep `DATABASE_URL` private and use a production PostgreSQL service. A static-only FTP plan cannot support the voting, nomination, admin APIs, file uploads, or PostgreSQL features in this application.

## Troubleshooting 403 / 404

- **403 at a directory URL:** the web server is not running the Next.js Node application. Configure the Node application/startup file or use a Node host.
- **404 on `/admin` or another app route:** confirm the domain points to the Node application and that the application is running; do not upload source files as static content.
- **Health check fails:** check the host's application logs and verify its `DATABASE_URL` points to a reachable PostgreSQL database.
- **IFrame is blank or blocked:** use the Node app's HTTPS origin for `src`, and ensure the hosting provider allows the app to be framed by `cscpo.org` (CSP / `X-Frame-Options`).
