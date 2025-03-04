# Configuration - Mend Renovate Community Edition for GitHub

## Create and Configure the GitHub App (bot)

Before running Mend Renovate, you need to provision it as an App on GitHub, and retrieve the ID + private key provided.

If you're running a self-hosted instance of GitHub Enterprise, it is suggested to name the app "Renovate" so that it shows up as easily recognizable as "renovate[bot]" in Pull Requests.
If you're running against `github.com` then the name Renovate is already taken by the hosted Mend Renovate app, so you will need something else like "YourCompany Renovate".

The App requires the following permissions:

- Repository permissions
  - Administration: Read-only
  - Checks: Read & write
  - Contents: Read & write
  - Issues: Read & write
  - Metadata: Read-only
  - Pull Requests: Read & write
  - Commit statuses: Read & write
  - Dependabot alerts: Read-only (optional)
  - Workflows: Read & write
- Organization permissions
  - Members: Read-only

The App should also subscribe to the following webhook events:

- Security Advisory
- Check run
- Check suite
- Issues
- Pull request
- Push
- Repository
- Status

Description, Homepage, User authorization callback URL, and Setup URL are all unimportant so you may set them to whatever you like.

The Mend Renovate webhook listener binds to port 8080 by default, however it will bind to `process.env.PORT` instead if that is defined.
Note: The Mend Renovate image takes care of exposing port 8080 of the container, so if you change this port then you will need to take care of any exposing/mapping of ports yourself.
In the [Docker Compose example config](https://github.com/mend/renovate-cc-ee/tree/main/examples/), the default port 8080 is used and then mapped to port 80 on the host.

For the Webhook URL field, point it to `/webhook` on port 80 (or whatever port you mapped to) of the server that you will run Mend Renovate on, e.g. http://1.2.3.4/webhook
Be sure to enter a webhook secret too.
If you don't care about the value, then enter 'renovate' as that is the default secret that the webhook handler process uses.

You can use the [Renovate icon](https://docs.renovatebot.com/assets/images/logo.png) for the app/bot if you desire.

## Configure Mend Renovate CE

### Mend Renovate environment variables

Mend Renovate requires configuration via environment variables in addition to Renovate OSS's regular configuration:

**`MEND_RNV_ACCEPT_TOS`**: Set this environment variable to `y` to consent to [Mend's Terms of Service](https://www.mend.io/terms-of-service/).

**`MEND_RNV_LICENSE_KEY`**: This should be the license key you obtained after registering at [https://www.mend.io/renovate-community/](https://www.mend.io/renovate-community/).

**`MEND_RNV_PLATFORM`**: Set this to `github`.

**`MEND_RNV_ENDPOINT`**: This is the API endpoint for your GitHub Enterprise installation. Required for GitHub Enterprise Server; not for GitHub.com. Include the trailing slash.

**`MEND_RNV_GITHUB_APP_ID`**: The GitHub App ID of the provisioned Renovate app on GitHub.

**`MEND_RNV_GITHUB_APP_KEY`**: A string representation of the private key of the provisioned Renovate app on GitHub. To insert the value directly into a Docker Compose environment variable, open the PEM file in a text editor and replace all new lines with "\n" so that the entire key is on one line. Alternatively, you can skip setting this key as an environment variable and instead mount it as a file to `/usr/src/app/renovate.private-key.pem`, as shown in the example Docker Compose file.

**`MEND_RNV_WEBHOOK_SECRET`**: Optional: Defaults to `renovate`

**`MEND_RNV_ADMIN_API_ENABLED`**: Optional: Set to 'true' to enable Admin APIs. Defaults to 'false'.

**`MEND_RNV_SERVER_API_SECRET`**: Required if Admin APIs are enabled.

**`MEND_RNV_SQLITE_FILE_PATH`**: Optional: Provide a path to persist the database. (eg. '/db/renovate-ce.sqlite', where 'db' is defined as a volume)

**`MEND_RNV_CRON_JOB_SCHEDULER`**: Optional: Accepts a 5-part cron schedule. Defaults to `0 * * * *` (i.e. once per hour exactly on the hour). This cron job triggers the Renovate bot against the projects in the SQLite database. If decreasing the interval then be careful that you do not exhaust the available hourly rate limit of the app on GitHub server or cause too much load.

**`MEND_RNV_CRON_APP_SYNC`**: Optional: Accepts a 5-part cron schedule. Defaults to `0 0,4,8,12,16,20 \* \* \*` (every 4 hours, on the hour). This cron job performs autodiscovery against the platform and fills the SQLite database with projects.

**`GITHUB_COM_TOKEN`**: A Personal Access Token for a user account on github.com (i.e. _not_ an account on your GitHub Enterprise instance). This is used for retrieving changelogs and release notes from repositories hosted on github.com and it does not matter who it belongs to. It needs only read-only access privileges. Note: This is required if you are using a self-hosted GitHub Enterprise or GitLab instance but should not be configured if your `RENOVATE_ENDPOINT` is `https://api.github.com`.

## Configure Renovate Core

The core Renovate OSS functionality can be configured using environment variables (e.g. `RENOVATE_XXXXXX`) or via a `config.js` file that you mount inside the Mend Renovate container to `/usr/src/app/config.js`.

**npm Registry** If using your own npm registry, you may find it easiest to update your Docker Compose file to include a volume that maps an `.npmrc` file to `/home/ubuntu/.npmrc`. The RC file should contain `registry=...` with the registry URL your company uses internally. This will allow Renovate to find shared configs and other internally published packages.

## Run Mend Renovate

You can run Mend Renovate from a Docker command line prompt, or by using a Docker Compose file. An example is provided below.

**Docker Compose File**: Renovate CE on GitHub

```yaml
version: "3.6"
services:
  renovate:
    image: ghcr.io/mend/renovate-ce:6.0.0-full
    restart: on-failure
    ports:
      - "80:8080" # Receive webhooks on port 80
    environment:
      # Optional: Define log level. Set to Debug for more verbose output
      # - LOG_LEVEL=DEBUG
      # Provide Mend License and accept Terms of Service
      - MEND_RNV_ACCEPT_TOS # Set to 'y' to accept the Mend Renovate Terms of Service
      - MEND_RNV_LICENSE_KEY # Set this to the key you received by email
      # Provide connection details for the GitHub App
      - MEND_RNV_PLATFORM=github
      - MEND_RNV_ENDPOINT # Required for GitHub Enterprise Server; not for GitHub.com. Include the trailing slash.
      - MEND_RNV_GITHUB_APP_ID # GitHub App ID
      - MEND_RNV_GITHUB_APP_KEY # GitHub App Key (PEM file). Alternatively mount as a volume below
      - MEND_RNV_WEBHOOK_SECRET # Optional: defaults to 'renovate'
      # Optional settings for Mend Renovate
      # - MEND_RNV_ADMIN_API_ENABLED # Optional: Set to 'true' to enable Admin APIs. Defaults to 'false'.
      # - MEND_RNV_SERVER_API_SECRET # Required if Admin APIs are enabled
      # - MEND_RNV_SQLITE_FILE_PATH=/db/renovate-ce.sqlite # Optional: Provide a path to persist the database. Needs 'db' volume defined (below).
      # - MEND_RNV_CRON_JOB_SCHEDULER # Optional Job enqueue schedule: defaults to '0 * * * *' (hourly, on the hour)
      # - MEND_RNV_CRON_APP_SYNC # Optional AppSync schedule: defaults to '0 0,4,8,12,16,20 * * *' (every 4 hours, on the hour)
      # Core Renovate settings
      # - GITHUB_COM_TOKEN # Personal Access Token for github.com (used for retrieving changelogs)
    # volumes:
      # Optional: You can use a file mount to persist the database between sessions
      # - "/tmp/db/:/db/" # Unix-style file mounting for the db
      # - "C:\\tmp/db/:/db/" # Windows-style file mounting for the db
```
