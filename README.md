# Cloudflare AllowMe

A simple yet powerful Node.js service / tool to automatically manage a list of IPs allowed in Cloudflare zone firewall. If you have a specific server that needs to be accessible from the internet, and you:

- Use Cloudflare as a DNS proxy / firewall
- Allow only IPs from Cloudflare to access your server from the outside
- Want a more granular, temporal access control via IP restrictions

Then this might be a good fit for you!

## Ultra quick start

1. Get your Cloudflare API token.
2. Set the nvironment variables:
    - `$ALLOWME_CF_ZONE` = your domain / zone
    - `$ALLOWME_CF_TOKEN` = your Cloudflare API token with the necessary permissions
    - `$ALLOWME_SERVER_SECRET` = your custom secret / password used to authenticate to the service
3. Run the `igoramadas/cloudflare-allowme` Docker image with the variables above.
4. Configure your mobile devices to ping the service every few hours to allow their current IP address on the Cloudflare firewall.

## Pre requisites

You should already have a zone (domain) registered with Cloudflare. If you don't, please follow [these steps](https://support.cloudflare.com/hc/en-us/articles/201720164-Creating-a-Cloudflare-account-and-adding-a-website).

This service itself requires minimal resources to run. You can spin it up on virtually any VM or cloud instance.

## Cloudflare preparation

### API token

Mandatory. First step is to create an API token for the service. If you already have a token with the necessary permissions and want to reuse it, you can skip to step 6.

1. Go to https://dash.cloudflare.com/profile/api-tokens.
2. Click on the "Create token" button, then proceed to "Custom token" > "Get started". [⧉](./docs/images/api-tokens.png)
3. Give the token a name (example: AllowMe), and the following permissions: [⧉](./docs/images/api-token-create.png)
    - Account > Account Filter Lists > Edit
    - Account > Account Settings > Read
    - Zone > Zone > Read
    - Zone > Zone WAF > Edit (optional)
4. Include the account and zone resources:
    - Include > _MY_ACCOUNT_NAME_
    - Include > Specific zone > _MY_ZONE.TLD_
5. Click "Continue to summary", then "Save token".
6. Copy the token value, it will be used as the `$ALLOWME_CF_TOKEN` variable.

### IP list

Optional. Next you'll have to define an IP rule list to manage the allowed IPs. By default, if you don't have any IP rule lists created on your Cloudflare account, the service will automatically create an "allowme" list for you, so you can skip these steps altogether. Otherwise if you want to do it manually:

1. Go to "Manage Account" > "Configurations" on the left sidebar of your dashboard. [⧉](./docs/images/sidebar-manage.png)
2. Select the "Lists" tab. [⧉](./docs/images/ip-lists.png)
3. Click on "Create new list", give it the name "allowme", content type "IP Address". [⧉](./docs/images/ip-list-create.png)
4. Click "Create" to finish.

If you already have an IP list that you want to reuse, you can simply grab its ID.

1. Go to "Manage Account" > "Configurations" on the sidebar of your dashboard. [⧉](./docs/images/sidebar-manage.png)
2. Select the "Lists" tab.
3. Click on "Edit" next to the list name.
4. Get the list ID from the URL, to be used as the `$ALLOWME_CF_LISTID` variable: [⧉](./docs/images/ip-list-edit.png)
    - Example: https://dash.cloudflare.com/account123/configurations/lists/LIST_ID

### Firewall rule

Optional. Pretty much like the IP list, the service can automatically create the firewall rule for you, but only if you have not specified a `$ALLOWME_CF_LISTID` variable manually. If you have specified it, then follow the steps:

1. Go to the zone dashboard on Cloudflare.
2. On the left sidebar, open "Security" > "WAF" (previously called Firewall Rules). [⧉](./docs/images/firewall.png)
3. Click on the "Create firewall rule" button.
4. Give it a name and the following properties:
    - Filter: "IP Source Address", "is in list", "allowme" (or the name of the list you have created manually)
    - Action: "Allow"
5. Click "Deploy" to save.

## Service configuration

The service is fully configured via environment variables, either directly or via a `.env` file. Variables marked with an asterisk * are mandatory.

| VARIABLE | TYPE | DETAILS |
| --- | --- | --- |
| **ALLOWME_CF_TOKEN** * |  string | Your Cloudflare API token. Mandatory. |
| **ALLOWME_CF_ACCOUNTID** | string | If you have multiple accounts, you can set the ID of the correct account here. If unset, the service will use the main account. The account ID can be taken from your dashboard URL, it's the token string right after dash.cloudflare.com/. |
| **ALLOWME_CF_ZONE** * | string | The zone which should be updated, for example "mydomain.com". |
| **ALLOWME_CF_LISTID** | string | Optional. The IP list ID, in case you don't want to have a dedicated "allowme" list. You can get the list ID from the URL of its edit page. If set, you'll have to configure the firewall rule manually (see the "Firewall rule" section above). |
| | | |
| **ALLOWME_SERVER_PORT** | number | Web server HTTP port. Defaults to "8080". |
| **ALLOWME_SERVER_SECRET** * | string | The secret / token that you should pass to the service via the Authorization (Bearer) header, or as the password on the Basic Auth prompt. Mandatory. |
| **ALLOWME_SERVER_USER** | string | Username to be used on the Basic Auth prompt (see below). Defaults to "allowme". |
| **ALLOWME_SERVER_PROMPT** | boolean | Optional, set to false to disable the Basic Auth prompt so only an Authorization (Bearer) header is accepted. |
| **ALLOWME_SERVER_TRUSTPROXY** | boolean | Optional, set to false to disable parsing the client IP from _X-Forwarded-For_ headers. |
| **ALLOWME_SERVER_HOME** | string | Optional, full URL to where users should be redirect if they git the root / path of the service. If missing the https://, this will be treated as text and that text will be displayed instead. Default is a redirection to the GitHub repo. |
| | | |
| **ALLOWME_IP_MAXAGE** | number | How long (in minutes) IPs should stay in the allowed list. Defaults to 1440 (1 day). Set to 0 to disable auto removing IPs. |
| **ALLOWME_IP_BLOCKINTERVAL** | number | How long (in minutes) IPs should be blocked in case of repeated authentication failures. Defaults to 60 minutes (1 hour). Set to 0 to disable blocking. |
| **ALLOWME_IP_DENYCOUNT** | number | How many times can an IP fail to authenticate before getting blocked. Defaults to 5. Setting to 0 will block IPs on their first failed auth. |
| | | |
| **ALLOWME_LOG_LEVEL** | none, error, info | Console logging level. Set to "none" to fully disable logging, or "error" to only log errors and warnings. Defaults to "info", which logs everything. |

#### Sample .env file

```
ALLOWME_CF_TOKEN=abc123abc123999000
ALLOWME_CF_ZONE=devv.com
ALLOWME_SERVER_PORT=80
ALLOWME_SERVER_SECRET=mysecret
ALLOWME_SERVER_PROMPT=false
```

## Running it with Docker

The easiest way to get this tool up and running is using the official Docker image:

```
$ docker pull igoramadas/cloudflare-allowme

$ docker run -it --name cloudflare-allowme \
             -p 80:8080 \
             -e ALLOWME_CF_TOKEN=MY_API_TOKEN \
             -e ALLOWME_CF_ZONE=MYDOMAIN.COM \
             -e ALLOWME_SERVER_SECRET=MY_SUPER_SECRET_KEY
             igoramadas/cloudflare-allowme
```

## Running it directly with Node.js

First, make sure you have all dependencies installed:

```
$ npm install --production
```

Then (on the root of the application) to start the service it's as simple as:

```
$ npm start
```

If you choose to have it running directly on your environment, it's highly recommended to use a process manager, for example [pm2](https://www.npmjs.com/package/pm2):

```
$ npm install pm2 -g
$ pm2 start lib/index.js
```

## Endpoints

| Method | Path | Details |
| - | - | - |
| GET | **/** | Redirect the user or display a text, depending on the `$ALLOWME_SERVER_HOME` variable |
| GET | **/allow** | Add the client IP to the allow list
| GET | **/block** | Remove the client IP from the allow list

## A few more tidbits

### Securing the endpoint with HTTPS

This service runs on HTTP only. You could use Cloudflare itself to have it running with HTTPS, or a self-hosted reverse proxy running next to the service instance: [nginx](https://docs.nginx.com/nginx/admin-guide/web-server/reverse-proxy/), [caddy](https://caddyserver.com/), [traefik](https://traefik.io/). If you decide to go with Caddy, have a look on [this other project](https://github.com/igoramadas/docker-caddy-cloudflare) of mine.

If you decide to put this service behind Cloudflare, you can add a few extra security constraints using its firewall. For instance, restricting access to the service only to specific User Agents, if you know what those user agents are beforehand.

### Automating requests on your mobile

If you have an Android device, you can use automation applications like [Tasker](https://tasker.joaoapps.com/) or [Automate](https://llamalab.com/automate/) to automatically call the service endpoint when your connection state changes (ie. connect or disconnect from Wifi).

I don't have iOS devices so I can test it for myself, but I think [Shortcuts](https://support.apple.com/en-gb/guide/shortcuts/welcome/ios) is your best bet.

If you decide to go full-automated then the `$ALLOWME_SERVER_PROMPT` can be set to "false".
