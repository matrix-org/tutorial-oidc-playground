# OIDC Tutorial Playground

**DO NOT USE THIS SETUP IN PRODUCTION**, it is only meant to setup a playground
for administrators who want to try a migration from password-based
authentication to OpenID Conect based authentication in a **disposable**
environment.

This playground environment allows administrators to play with the Synapse admin
API to retrieve all users from Synapse, and with Authentik's API to retrieve all
users from the OpenID provider. They can reconciliate the accounts, and then use
Synapse's admin API to update the accounts in Synapse so they match their OpenID
counterpart.

## What does it do

This playbook installs podman, deploys Synapse and Authentik (an OpenID
provider) as systemd services behind a reverse-proxy, and pre-populates them
with a few users.

| User            | Username in Synapse | Email in Synapse    | Username in Authentik | Email in Authentik    |
|-----------------|---------------------|---------------------|-----------------------|-----------------------|
| Stephen Baxter  | sbaxter             | sbaxter@example.com | sbaxter               | sbaxter@example.com   |
| Douglas Adams   | doug                | dadams@example.com  | dadams                | dadams@example.com    |
| Isaac Asimov    | iasimov             | iasimov@example.com |                       |                       |
| Nora K. Jemisin |                     |                     | nkjemisin             | nkjemisin@example.com |

## Setting-up the base environment

### Pre-requisites

This playbook has only been tested on Rocky Linux 9, and should work on most
RHEL derivatives. This narrow scope is on-purpose to limit the maintenance cost
in the long run, given it's a playground environment that is expected to run on
a fresh server. It assumes you can ssh as root on this server (remember it's a
playground) and have already copied your ssh key.

You need a freshly deployed server open on the Internet and you need a domain
name. Assuming you're going to use playground.example.com for your base domain,
you need to add the following records:
 * A record from playground.example.com to your server
 * CNAME record from matrix.playground.example.com to playground.example.com 
 * CNAME record from login.playground.example.com to playground.example.com

### Filling the blanks

To get this project running, clone this repostory, open `inventory/playground`
and remplace `playground.example.com` with your actual domain name.

Rename the `playground.example.com` directory inside `inventory/host_vars/` to
your actual domain name, and open `inventory/host_vars/your_domain/vars.yml`

Fill it according to comments inside. Remember to leave the `client_id` and
`client_secret` fields commented for now.

### Deploying the services

Once everything is ready, use the following command to deploy Synapse, Authentik
and the reverse-proxy on your setup.

```
ansible-playbook -i inventory/playground sandbox.yml --tags install
```

After executing the playbook, head to https://login.yourdomain and you should
see Authentik's login page. Head to https://matrix.yourdomain and you should see
Synapse's welcome page. If get a 404, traefik might have hard a hard time
routing traffic to Synapse's container and needs a restart. You can do so with:

```
ansible-playbook -i inventory/playground sandbox.yml --tags restart-traefik
```

### Adding users

To match what could have happened in a production setup, we will populate
Synapse and Authentik according to the table above before allowing Synapse to
use Authentik.

Run the following command to populate both services:

```
ansible-playbook -i inventoy/playground sandbox.yml --tags populate
```

## Connecting Synapse to Authentik

To allow Synapse to use Authentik, we need to create an [Authentik
Application](https://goauthentik.io/docs/applications) for it, based on an
[Authentik OAuth2 Provider](https://goauthentik.io/docs/providers/oauth2/). The
application MUST be named `synapse`.

When doing so, retrieve the `client_id` and `client_secret` Authentik generated
for synapse, add them to your `inventory/host_vars/your_domain/vars.yml` and
uncomment the lines containing it.

Authentik is now willing to listen to Synapse, we must now tell Synapse to use
Authentik. Now the `client_id` and `client_secret` variables are uncommented,
the playbook will update Synapse's configuration. If you want to have a closer
look at what is happening behind the scenes, open
`roles/synapse/templates/homeserver.yaml` and inspect the `oidc_providers`
section. The full details of the configuration can be found in [Synapse's
Configuration
Manual](https://matrix-org.github.io/synapse/latest/usage/configuration/config_documentation.html#oidc_providers).

Redeploy synapse with the following command:

```
ansible-playbook -i inventory/playground sandbox.yml --tags synapse
```

Here again, traefik might need a restart if https://matrix.yourdomain returns a 404:

```
ansible-playbook -i inventory/playground sandbox.yml --tags restart-traefik
```

If you go to an Element instance where you're not logged in (e.g.
https://staging.element.io) and change the homeserver to your domain, Element
should offer you to connect either by login and password or to continue with "My
Company", which is how this playbook calls Authentik.