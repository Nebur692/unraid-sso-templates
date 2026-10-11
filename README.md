<div align="center">

# Unraid SSO Templates

*Working Unraid Community Applications templates for a self-hosted SSO stack: ZITADEL v4 (core + the login UI it actually needs) and oauth2-proxy*

![Release](https://img.shields.io/github/v/release/Nebur692/unraid-sso-templates?label=release&color=blue)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![GitHub Sponsors](https://img.shields.io/badge/Sponsor-30363D?logo=githubsponsors&logoColor=EA4AAA)](https://github.com/sponsors/Nebur692)
[![Ko-fi](https://ko-fi.com/img/githubbutton_sm.svg)](https://ko-fi.com/nebur69265723)
[![PayPal](https://img.shields.io/badge/PayPal-donate-00457C?logo=paypal&logoColor=white)](https://paypal.me/0SkillS)

🇬🇧 [English](#english) · 🇪🇸 [Español](#español)

</div>

---

## English

### ✨ Why this exists

The only ZITADEL template previously available on Unraid Community Applications
([jcesclapez/unraid-templates](https://github.com/jcesclapez/unraid-templates)) was written for
ZITADEL's older, single-container architecture. Since **ZITADEL v4**, the interactive login pages
(username/password, MFA, WebAuthn...) were split into their own service, **zitadel-login**. Deploy
only the core container like that template does, and every login attempt fails with a silent 404 on
`/ui/v2/login/login` — the container itself reports healthy, so this is easy to mistake for a
reverse-proxy problem when it's actually a missing second container.

This repository provides **two templates that work together**:

- **`zitadel`** — the core/API container, corrected and completed with every variable ZITADEL v4
  actually needs to bootstrap the login-client token on first start (`ZITADEL_FIRSTINSTANCE_*`,
  `ZITADEL_DEFAULTINSTANCE_FEATURES_LOGINV2_*`, `ZITADEL_OIDC_DEFAULTLOGINURLV2`, etc.) — none of
  which existed in the previous template.
- **`zitadel-login`** — the missing login UI container. There was no Community Applications template
  for this at all before this repository.

Both were built directly from ZITADEL's own [reference docker-compose
files](https://github.com/zitadel/zitadel/tree/main/deploy/compose), not guessed — the variable
names and required values are taken straight from there.

### 🧭 Deployment order

**`zitadel` (core) always goes first, `zitadel-login` always second — never the other way round.**
The core container writes a login token to a shared file the very first time it starts; the login
container reads that same file to authenticate itself, so it has nothing to read if it starts
first (it will just sit unhealthy). Postgres has to exist before either of them.

```
1. PostgreSQL (external, not part of this repo)
2. zitadel        ← wait until its log says "setup completed"
3. zitadel-login  ← only start this once step 2 finished
```

### 📦 Installation

The old dedicated "Template repositories" field under Docker settings was deprecated years ago in
favor of Community Applications, so it isn't present in current Unraid (confirmed on 7.3.2) — don't
go looking for it. Unraid actually discovers templates from **any XML file placed in
`/boot/config/plugins/dockerMan/templates-user/`** — that's the exact same folder/mechanism it uses
to remember every container you've ever configured from the GUI, so dropping a file in there is a
completely normal, version-independent way to make a template show up.

1. **Set up an external PostgreSQL 14+ container first** (not included — any standard Postgres
   template works). Leave `POSTGRES_USER` at its default (`postgres`) unless you have a specific
   reason to change it — see the warning in the `zitadel` template's *Database Admin Username*
   field below if you do.
2. Download both template files into that folder, e.g. via the Unraid terminal:
   ```bash
   curl -o /boot/config/plugins/dockerMan/templates-user/unraid-sso-templates_zitadel.xml \
     https://raw.githubusercontent.com/Nebur692/unraid-community-apps/main/templates/zitadel.xml
   curl -o /boot/config/plugins/dockerMan/templates-user/unraid-sso-templates_zitadel-login.xml \
     https://raw.githubusercontent.com/Nebur692/unraid-community-apps/main/templates/zitadel-login.xml
   curl -o /boot/config/plugins/dockerMan/templates-user/unraid-sso-templates_oauth2-proxy.xml \
     https://raw.githubusercontent.com/Nebur692/unraid-community-apps/main/templates/oauth2-proxy.xml
   ```
   (Use any filename you like — Unraid doesn't care, it scans the whole folder.)
3. **Docker → Add Container**, open the **Template** dropdown at the top — `zitadel` and
   `zitadel-login` now appear in the list, pre-filled with every field described below. Fill in:
   Master Key (32 random characters, generate once and keep it — you cannot change it later
   without losing encrypted data), External Domain, the database connection fields, and the four
   `*_LOGINURLV2`/`LOGINV2_BASEURI` variables (derived from your External Domain/Port).
   *(If you'd rather not touch the terminal, you can instead pick Repository `ghcr.io/zitadel/zitadel:latest`
   manually and add each variable/port/path yourself, reading
   [`zitadel.xml`](https://github.com/Nebur692/unraid-community-apps/blob/main/templates/zitadel.xml) as reference — slower, but works the same.)*
4. Start it, and confirm the container's log shows `setup completed` with no errors before
   continuing.
5. **Docker → Add Container** again, pick the `zitadel-login` template from the dropdown. Fill in
   `ZITADEL_API_URL` (the `zitadel` container's internal IP and port, e.g.
   `http://192.168.1.50:8080`) and `CUSTOM_REQUEST_HEADERS` (your domain). Point its **Bootstrap
   Volume** at the exact same host path you used for the `zitadel` container — this is how it
   reads the token the core container wrote on its first start.
6. Start `zitadel-login` and confirm it comes up healthy.

### 🖥️ First login: two different web addresses, not one

This trips up almost everyone the first time, technical or not, so read this before you go looking
for a "dashboard" and can't find it.

There are **two completely separate pages**, and mixing them up is the most common point of
confusion:

| Address | What it actually is |
|---|---|
| `https://your-domain/ui/v2/login` | Just the **sign-in door**: type your email, type your password. That's all it does. |
| `https://your-domain/ui/console` | The actual **admin panel** — where you manage everything: users, organizations, and the connections to Zabbix/WordPress/etc. |

If you type your address into the browser and it takes you straight to the sign-in door, log in
normally — but once you're done, **you'll land on a plain "you're signed in" page that goes
nowhere by itself.** That's expected. It is *not* the admin panel, and there is no button on it
that takes you there.

**To reach the actual admin panel, type this address directly into your browser:**
```
https://your-domain/ui/console
```
(replacing `your-domain` with your own External Domain). Since you're already signed in, it should
open straight away, or ask you to confirm the account once. This is where you'll do everything
described in this README from now on: create the SSO connections to Zabbix, WordPress, etc.

**Default first-time login**, if you haven't changed it yet:
- Username: `zitadel-admin@zitadel.<your External Domain>` — note the extra `zitadel.` piece
  glued in front of your own domain, that's not a typo, it's ZITADEL's own default naming.
- Password: `Password1!`

### ⚙️ Configuration reference

> **Why do External Domain/Port/Secure and the four `*URLV2` fields repeat the same information?**
> ZITADEL's own reference docker-compose builds those four URLs *automatically* from the
> domain/port/scheme, using docker-compose's own variable-substitution syntax. Plain Unraid
> templates have no equivalent — every `<Config>` field is independent static text, there's no way
> for one field to reference another's value. So you fill in the same domain/port/scheme by hand,
> pasted into 4 fixed patterns. Given `External Domain=sso.example.com`, `Port=443`, `Secure=true`:
> ```
> Login V2 Base URI:         https://sso.example.com:443/ui/v2/login/
> OIDC Default Login URL V2:  https://sso.example.com:443/ui/v2/login/login?authRequest=
> OIDC Default Logout URL V2: https://sso.example.com:443/ui/v2/login/logout?post_logout_redirect=
> SAML Default Login URL V2:  https://sso.example.com:443/ui/v2/login/login?samlRequest=
> ```
> Just substitute your own domain/port/scheme into those same 4 patterns — nothing to reason
> about, it's mechanical.

#### `zitadel` (core)

| Variable | What it's actually for | Example |
|---|---|---|
| Master Key | Encryption key for secrets stored in the database. Set once, **never** change it afterwards or you lose access to everything encrypted with it. Exactly 32 characters. **⚠️ Also paste this exact same value into "Post Arguments" (Extra Parameters, near the bottom of Add Container) replacing the placeholder in `--masterkey "..."` — a known ZITADEL bug means the environment variable alone is not reliably read, and the container will crash-loop with `no master key provided` if you skip this.** | `openssl rand -base64 24 \| cut -c1-32` |
| External Domain | The domain your *users* type into their browser — must match your reverse proxy's configured domain. | `sso.example.com` |
| External Port | The *public-facing* port (what's after the domain from the outside), not the container's internal port. Almost always `443`. | `443` |
| External Secure (HTTPS) | Tell ZITADEL whether the outside world reaches it over HTTPS. Keep `true` unless you're doing a plain-HTTP local test. | `true` |
| TLS Enabled (container-internal) | Whether ZITADEL itself should terminate HTTPS *inside the container*. Leave `false` — your reverse proxy (NPM/Traefik/Caddy) should be the one handling certificates. | `false` |
| Database Postgres Host/Port | Where to find your external Postgres. | `192.168.1.10`, `5432` |
| Database Name | Which database inside Postgres ZITADEL uses. | `zitadel` |
| Database Admin Username/Password | Credentials with rights to create the app role and database on first start. **If your Postgres container's `POSTGRES_USER` isn't the default `postgres`, put that custom username here instead** — see Troubleshooting. | `postgres` |
| Database App Username/Password | The day-to-day, lower-privilege credentials ZITADEL creates and uses afterwards. | `zitadel` |
| Require Password Change On First Login | Whether the built-in default admin account (`zitadel-admin@zitadel.<domain>` / `Password1!`) must set a new password the first time it logs in. Recommended: `true`. | `true` |
| Login Client PAT Path (bootstrap) | **Don't touch.** The file path, inside the shared Bootstrap Volume, where ZITADEL writes the access token `zitadel-login` needs to function. | `/zitadel/bootstrap/login-client.pat` |
| Login Client Machine Username/Name | **Don't touch.** Internal service-account identity ZITADEL creates for the login UI to authenticate as. | `login-client` |
| Login Client PAT Expiration Date | **Don't leave empty.** A valid RFC3339 date the login-client token is valid until. Without this, ZITADEL silently skips writing the token file entirely — no error anywhere, `zitadel-login` just never works. The default is ~100 years out. | `2124-01-01T00:00:00Z` |
| Login V2 Required / Base URI | `Required` is always `true` in v4 (kept visible, not meant to be edited). `Base URI` is the full public URL where the login pages live — build it from your own External Domain/Port/Secure above. | `https://sso.example.com/ui/v2/login/` |
| OIDC Default Login/Logout URL V2 | Where OIDC clients get redirected to sign in / sign out. Same domain as above, different suffix. | `.../ui/v2/login/login?authRequest=`, `.../ui/v2/login/logout?post_logout_redirect=` |
| SAML Default Login URL V2 | Same idea, for SAML clients (e.g. Zabbix). Only needed if you'll use SAML. | `.../ui/v2/login/login?samlRequest=` |
| Bootstrap Volume | A folder shared with `zitadel-login` (mounted read-only there) so it can read the token above. Both templates must point at the **exact same** host path. | `/mnt/user/appdata/zitadel/bootstrap` |
| Web UI / API Port | Internal port. Put your reverse proxy in front — don't expose straight to the internet. | `8080` |
| HTTP Client Deny List (SSRF protection) | ZITADEL refuses to call Action Target endpoints on private IP ranges by default — **this includes your whole home LAN** (`192.168.0.0/16`), so creating a Target pointing at any other container on your network fails with `Errors.Target.DeniedURL`. The default value here is ZITADEL's own stock deny list with just `192.168.0.0/16` removed — everything else (loopback, link-local, cloud metadata endpoints like `169.254.169.254`) stays blocked. Only add more ranges if you understand you're loosening SSRF protection. | see Default column |

#### `zitadel-login`

| Variable | What it's actually for | Example |
|---|---|---|
| ZITADEL API URL | How this container reaches the `zitadel` core container internally — its LAN/container IP and port, **not** your public domain. | `http://192.168.1.50:8080` |
| Public Base Path | The URL path this login UI is served under. Must match what your reverse proxy routes here. Leave as-is unless you know why you'd change it. | `/ui/v2/login` |
| Service User Token File | **Don't touch.** Where inside this container it reads the token `zitadel` wrote. Must match the Bootstrap Volume path below. | `/zitadel/bootstrap/login-client.pat` |
| Custom Request Headers | Tells the login app what Host/protocol to think it's being served as, since it's sitting behind a reverse proxy that already terminated HTTPS. Without this, cookies/redirects can break. | `Host:sso.example.com,X-Forwarded-Proto:https` |
| Bootstrap Volume | Must be the **exact same host path** as the Bootstrap Volume in the `zitadel` template — this is how the token gets shared between the two containers. | `/mnt/user/appdata/zitadel/bootstrap` |
| Login UI Port | Internal port. Your reverse proxy routes `/ui/v2/login/*` here (see below). | `3000` |

### ⚙️ Reverse proxy configuration (required)

Your reverse proxy needs to send two different URL patterns to two different containers:

| Path | Goes to |
|---|---|
| `/ui/v2/login/*` | `zitadel-login` container, port `3000` |
| everything else (`/ui/console`, `/oauth/*`, `/oidc/*`, `/api/*`...) | `zitadel` container, port `8080` |

**Nginx Proxy Manager example:** on the proxy host pointing at your `zitadel` container (port
8080) as usual, add a **Custom Location**:
- Location: `/ui/v2/login`
- Forward Hostname/IP: your `zitadel-login` container's IP
- Forward Port: `3000`

Without this, the OIDC/SAML login flow will redirect correctly but land on a 404 the moment it
needs to show the actual sign-in form.

### 🩹 Troubleshooting

- **Container `zitadel` restart-looping, log says `password authentication failed for user
  "postgres"`**: your Postgres container was created with a custom `POSTGRES_USER` (not the
  default `postgres`), which *replaces* the default superuser rather than adding a second one. Put
  that same custom username in the `zitadel` template's *Database Admin Username* field instead of
  `postgres`.
- **Login page shows a 404 on `/ui/v2/login/login`**: `zitadel-login` isn't running, isn't
  reachable, or your reverse proxy isn't routing that path to it — see the section above.
- **`zitadel-login` can't read the token / stays unhealthy**: its Bootstrap Volume path must be
  the *exact* same host path as the `zitadel` container's Bootstrap Volume, and `zitadel` must have
  completed its first start (which is when the token file is written) before `zitadel-login` starts.
- **`zitadel` logs `setup completed` with no errors, but the Bootstrap Volume folder stays
  completely empty (no `login-client.pat` ever appears)**: two independent causes, check both.
  1. **Folder permissions.** ZITADEL runs internally as UID 1000. If Unraid auto-created the
     Bootstrap Volume folder on first start, it's likely owned by `root` or `nobody` without write
     access for UID 1000 — the token write fails completely silently, nothing is logged anywhere.
     Fix: on the Unraid host, `chown -R 1000:1000 <your bootstrap path>`.
  2. **Missing PAT Expiration Date.** If *Login Client PAT Expiration Date* was left empty, ZITADEL
     silently skips creating the token altogether, again with zero log output. Fix: set it (see the
     reference table above).
  Either way, **restarting the container alone will not fix this** — the entire first-instance
  bootstrap (creating the admin user, the org, the login-client service account, and its token)
  only ever runs once, the very first time against a genuinely empty database. If it already ran
  once and failed partway (e.g. wrote the DB rows but not the file), you must wipe the Postgres
  database clean and restart `zitadel` again for it to retry from scratch — fixing the
  permissions/variable first, then just restarting, changes nothing.
- **Creating an Actions V2 Target fails with `[invalid_argument] Errors.Target.DeniedURL`**: by
  default ZITADEL refuses to call webhook endpoints on private IP ranges as an SSRF protection —
  this includes your entire home LAN (`192.168.0.0/16`). If you're building something that calls
  back into your own network (e.g. a custom login-approval webhook), set the *HTTP Client Deny
  List* variable above — its default value here already removes just `192.168.0.0/16` from
  ZITADEL's stock list, keeping every other protected range (loopback, link-local, cloud metadata
  endpoints) intact. This variable only takes effect on the `zitadel` core container, and requires
  recreating it (a plain restart won't pick up a newly added environment variable).

### 🔐 `oauth2-proxy` — putting that login in front of anything else

The third template in this repository isn't part of ZITADEL: it's
[**oauth2-proxy**](https://oauth2-proxy.github.io/oauth2-proxy/), the piece that lets you reuse the
identity provider you just deployed to protect web apps that have **no login of their own** — a
router page, a monitoring dashboard, an old admin panel. Your reverse proxy asks oauth2-proxy on
every request whether the visitor is signed in; if not, the visitor goes to your provider and only
comes back once authenticated.

A template for oauth2-proxy already existed on Community Applications and it works, but it exposes
**no settings at all** — every option has to go into a config file by hand. This one exposes each
option as its own field, with an explanation of what it does and why it matters.

**One container, many sites.** Leave *Redirect URL* as a path (`/oauth2/callback`) instead of a full
address: oauth2-proxy then derives the absolute URL from the host of each request, so the same
container can guard any number of domains. Give *Cookie domains* a leading dot (`.example.com`) and
one login covers every protected site under it.
*(Do not use the `--relative-redirect-url` flag to achieve this — it sends a literally relative
`redirect_uri`, which standards-compliant providers reject.)*

**Nginx Proxy Manager**, which has no `auth_request` field in its UI, still supports this: anything
you put in a host's *Advanced* tab is inserted at server level, and NPM skips its own default
`location /` as soon as your config declares one. So this is all it takes, per protected host:

```nginx
location /oauth2/ {
    proxy_pass         http://OAUTH2_PROXY_IP:4180;
    proxy_set_header   Host              $host;
    proxy_set_header   X-Real-IP         $remote_addr;
    proxy_set_header   X-Forwarded-For   $proxy_add_x_forwarded_for;
    proxy_set_header   X-Forwarded-Proto $x_forwarded_proto;
}

location = /oauth2/auth {
    internal;
    proxy_pass              http://OAUTH2_PROXY_IP:4180;
    proxy_set_header        Host              $host;
    proxy_set_header        X-Real-IP         $remote_addr;
    proxy_set_header        X-Forwarded-For   $proxy_add_x_forwarded_for;
    proxy_set_header        X-Forwarded-Proto $x_forwarded_proto;
    proxy_set_header        Content-Length    "";
    proxy_pass_request_body off;
}

location @oauth2_signin {
    return 302 https://$host/oauth2/start?rd=$request_uri;
}

location / {
    auth_request /oauth2/auth;
    error_page 401 = @oauth2_signin;

    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection $http_connection;
    proxy_http_version 1.1;

    include conf.d/include/proxy.conf;
}
```

Register `https://that-site/oauth2/callback` as a redirect URI at your provider for **each** host
you protect, and set the visitor's e-mail in the allow-list file. Two things worth knowing before
you rely on it:

- **Everyone who gets in, gets into everything** behind the same instance. For a different audience
  (family, guests) run a second container with its own cookie name, its own application at the
  provider and its own allow-list.
- **Don't put it in front of machine traffic** — monitoring pollers, mobile apps, webhooks, APIs.
  It is a gate for browsers; anything non-interactive will simply be turned away at the redirect.

### 💙 Support

None of this would be possible without the community's support. If this project has been useful to
you, consider supporting it via [GitHub Sponsors](https://github.com/sponsors/Nebur692),
[Ko-fi](https://ko-fi.com/nebur69265723) or [PayPal](https://paypal.me/0SkillS) — every bit helps
keep it maintained.

### ⚠️ Disclaimer

Not affiliated with, endorsed by, or associated with ZITADEL or ZITADEL GmbH. "ZITADEL" is a
trademark of its respective owner. Not affiliated with jcesclapez/unraid-templates beyond crediting
it as the reason this repository exists.

### 📜 License

[MIT](LICENSE)

---

## Español

### ✨ Por qué existe esto

La única plantilla de ZITADEL antes disponible en Community Applications de Unraid
([jcesclapez/unraid-templates](https://github.com/jcesclapez/unraid-templates)) estaba pensada para
la arquitectura antigua de ZITADEL, de un solo contenedor. Desde **ZITADEL v4**, las páginas de
login interactivas (usuario/contraseña, MFA, WebAuthn...) se separaron en su propio servicio,
**zitadel-login**. Si despliegas solo el contenedor principal como hacía esa plantilla, cualquier
intento de login falla con un 404 silencioso en `/ui/v2/login/login` — el propio contenedor se
reporta como sano, así que es fácil confundirlo con un problema del proxy inverso cuando en
realidad falta un segundo contenedor.

Este repositorio aporta **dos plantillas que funcionan juntas**:

- **`zitadel`** — el contenedor principal/API, corregido y completado con todas las variables que
  ZITADEL v4 necesita de verdad para generar el token del login-client en el primer arranque
  (`ZITADEL_FIRSTINSTANCE_*`, `ZITADEL_DEFAULTINSTANCE_FEATURES_LOGINV2_*`,
  `ZITADEL_OIDC_DEFAULTLOGINURLV2`, etc.) — ninguna de las cuales existía en la plantilla anterior.
- **`zitadel-login`** — el contenedor de login que faltaba. No existía ninguna plantilla de
  Community Applications para esto antes de este repositorio.

Ambas se construyeron directamente a partir de los [docker-compose de referencia oficiales de
ZITADEL](https://github.com/zitadel/zitadel/tree/main/deploy/compose), no por suposición — los
nombres de variable y los valores requeridos salen de ahí directamente.

### 🧭 Orden de despliegue

**`zitadel` (principal) siempre va primero, `zitadel-login` siempre segundo — nunca al revés.** El
contenedor principal escribe un token de login en un fichero compartido la primerísima vez que
arranca; el contenedor de login lee ese mismo fichero para autenticarse, así que no tiene nada que
leer si arranca antes (se quedará simplemente unhealthy). Postgres tiene que existir antes que
cualquiera de los dos.

```
1. PostgreSQL (externo, no incluido en este repo)
2. zitadel        ← espera a que su log diga "setup completed"
3. zitadel-login  ← arráncalo solo cuando el paso 2 haya terminado
```

### 📦 Instalación

El antiguo campo dedicado "Template repositories" de los ajustes de Docker quedó obsoleto hace años
en favor de Community Applications, así que no está presente en el Unraid actual (confirmado en
7.3.2) — no lo busques. Unraid en realidad descubre las plantillas a partir de **cualquier fichero
XML que exista en `/boot/config/plugins/dockerMan/templates-user/`** — es exactamente la misma
carpeta/mecanismo que usa para recordar cada contenedor que has configurado alguna vez desde la
interfaz, así que dejar un fichero ahí es una forma completamente normal e independiente de la
versión de hacer que una plantilla aparezca.

1. **Monta primero un contenedor externo de PostgreSQL 14+** (no incluido — cualquier plantilla
   estándar de Postgres vale). Deja `POSTGRES_USER` en su valor por defecto (`postgres`) salvo que
   tengas una razón concreta para cambiarlo — mira el aviso en el campo *Database Admin Username*
   de la plantilla `zitadel` más abajo si lo haces.
2. Descarga ambas plantillas a esa carpeta, por ejemplo desde la terminal de Unraid:
   ```bash
   curl -o /boot/config/plugins/dockerMan/templates-user/unraid-sso-templates_zitadel.xml \
     https://raw.githubusercontent.com/Nebur692/unraid-community-apps/main/templates/zitadel.xml
   curl -o /boot/config/plugins/dockerMan/templates-user/unraid-sso-templates_zitadel-login.xml \
     https://raw.githubusercontent.com/Nebur692/unraid-community-apps/main/templates/zitadel-login.xml
   curl -o /boot/config/plugins/dockerMan/templates-user/unraid-sso-templates_oauth2-proxy.xml \
     https://raw.githubusercontent.com/Nebur692/unraid-community-apps/main/templates/oauth2-proxy.xml
   ```
   (El nombre del fichero da igual — Unraid escanea toda la carpeta.)
3. **Docker → Add Container**, abre el desplegable **Template** de arriba — `zitadel` y
   `zitadel-login` ya aparecen en la lista, con todos los campos descritos abajo precargados.
   Rellena: Master Key (32 caracteres aleatorios, genéralo una vez y guárdalo — no se puede cambiar
   después sin perder datos cifrados), External Domain, los campos de conexión a la base de datos,
   y las cuatro variables `*_LOGINURLV2`/`LOGINV2_BASEURI` (derivadas de tu External Domain/Port).
   *(Si prefieres no tocar la terminal, también puedes elegir Repository
   `ghcr.io/zitadel/zitadel:latest` a mano y añadir cada variable/puerto/ruta tú mismo, usando
   [`zitadel.xml`](https://github.com/Nebur692/unraid-community-apps/blob/main/templates/zitadel.xml) como referencia — más lento, pero funciona igual.)*
4. Arráncalo, y confirma que el log del contenedor muestra `setup completed` sin errores antes de
   continuar.
5. **Docker → Add Container** otra vez, elige la plantilla `zitadel-login` del desplegable.
   Rellena `ZITADEL_API_URL` (la IP y puerto internos del contenedor `zitadel`, p.ej.
   `http://192.168.1.50:8080`) y `CUSTOM_REQUEST_HEADERS` (tu dominio). Pon su **Bootstrap Volume**
   apuntando exactamente a la misma ruta de host que usaste en el contenedor `zitadel` — así es
   como lee el token que el contenedor principal escribió en su primer arranque.
6. Arranca `zitadel-login` y confirma que queda healthy.

### 🖥️ Primer inicio de sesión: dos direcciones web distintas, no una sola

Esto confunde a casi todo el mundo la primera vez, seas informático o no, así que lee esto antes
de ponerte a buscar un "panel" y no encontrarlo.

Hay **dos páginas completamente separadas**, y confundirlas es el lío más habitual:

| Dirección | Qué es realmente |
|---|---|
| `https://tu-dominio/ui/v2/login` | Solo la **puerta de entrada**: escribes tu correo, escribes tu contraseña. Eso es todo lo que hace. |
| `https://tu-dominio/ui/console` | El **panel de administración** de verdad — donde gestionas todo: usuarios, organizaciones, y las conexiones con Zabbix/WordPress/etc. |

Si escribes tu dirección en el navegador y te lleva directo a la puerta de entrada, inicia sesión
con normalidad — pero al terminar, **acabarás en una página sencilla de "ya has iniciado sesión"
que no te lleva a ningún sitio por sí sola.** Es normal. Esa página *no es* el panel de
administración, y no tiene ningún botón que te lleve ahí.

**Para llegar al panel de administración de verdad, escribe esta dirección directamente en el
navegador:**
```
https://tu-dominio/ui/console
```
(cambiando `tu-dominio` por tu propio External Domain). Como ya tienes sesión iniciada, debería
abrirse directamente, o pedirte confirmar la cuenta una vez. Aquí es donde harás a partir de ahora
todo lo que describe este README: crear las conexiones de SSO con Zabbix, WordPress, etc.

**Inicio de sesión por defecto**, si todavía no lo has cambiado:
- Usuario: `zitadel-admin@zitadel.<tu External Domain>` — fíjate en el trozo `zitadel.` extra
  pegado delante de tu propio dominio, no es un error tipográfico, es el nombre por defecto que
  pone el propio Zitadel.
- Contraseña: `Password1!`

### ⚙️ Referencia de configuración

> **¿Por qué External Domain/Port/Secure y los cuatro campos `*URLV2` repiten la misma
> información?** El docker-compose de referencia de Zitadel construye esas cuatro URLs
> *automáticamente* a partir del dominio/puerto/esquema, usando la propia sintaxis de sustitución
> de variables de docker-compose. Las plantillas normales de Unraid no tienen equivalente — cada
> campo `<Config>` es texto estático independiente, no hay forma de que un campo referencie el
> valor de otro. Así que rellenas a mano el mismo dominio/puerto/esquema, pegado en 4 patrones
> fijos. Con `External Domain=sso.example.com`, `Port=443`, `Secure=true`:
> ```
> Login V2 Base URI:          https://sso.example.com:443/ui/v2/login/
> OIDC Default Login URL V2:  https://sso.example.com:443/ui/v2/login/login?authRequest=
> OIDC Default Logout URL V2: https://sso.example.com:443/ui/v2/login/logout?post_logout_redirect=
> SAML Default Login URL V2:  https://sso.example.com:443/ui/v2/login/login?samlRequest=
> ```
> Solo hay que sustituir tu propio dominio/puerto/esquema en esos mismos 4 patrones — nada que
> razonar, es mecánico.

#### `zitadel` (principal)

| Variable | Para qué sirve de verdad | Ejemplo |
|---|---|---|
| Master Key | Clave de cifrado de los secretos guardados en la base de datos. Se pone una vez y **nunca** se cambia después, o pierdes acceso a todo lo cifrado con ella. Exactamente 32 caracteres. **⚠️ Pega también ese mismo valor en "Post Arguments" (Extra Parameters, cerca del final de Add Container) sustituyendo el placeholder en `--masterkey "..."` — un bug conocido de Zitadel hace que la variable de entorno sola no se lea de forma fiable, y el contenedor se reiniciará en bucle con `no master key provided` si te saltas esto.** | `openssl rand -base64 24 \| cut -c1-32` |
| External Domain | El dominio que escriben tus *usuarios* en el navegador — debe coincidir con el dominio configurado en tu proxy inverso. | `sso.example.com` |
| External Port | El puerto *de cara al público* (lo que va después del dominio, visto desde fuera), no el puerto interno del contenedor. Casi siempre `443`. | `443` |
| External Secure (HTTPS) | Le dice a Zitadel si el mundo exterior le llega por HTTPS. Déjalo en `true` salvo que sea una prueba local en HTTP plano. | `true` |
| TLS Enabled (container-internal) | Si Zitadel debe terminar el HTTPS él mismo *dentro del contenedor*. Déjalo en `false` — tu proxy inverso (NPM/Traefik/Caddy) es quien debe gestionar los certificados. | `false` |
| Database Postgres Host/Port | Dónde encontrar tu Postgres externo. | `192.168.1.10`, `5432` |
| Database Name | Qué base de datos dentro de Postgres usa Zitadel. | `zitadel` |
| Database Admin Username/Password | Credenciales con permisos para crear el rol y la base de datos de la app en el primer arranque. **Si el `POSTGRES_USER` de tu contenedor de Postgres no es el `postgres` por defecto, pon aquí ese usuario personalizado** — ver Solución de problemas. | `postgres` |
| Database App Username/Password | Las credenciales de uso diario, de menos privilegio, que Zitadel crea y usa después. | `zitadel` |
| Require Password Change On First Login | Si la cuenta de admin por defecto (`zitadel-admin@zitadel.<tu dominio>` / `Password1!`) debe fijar una contraseña nueva la primera vez que entra. Recomendado: `true`. | `true` |
| Login Client PAT Path (bootstrap) | **No tocar.** La ruta del fichero, dentro del Bootstrap Volume compartido, donde Zitadel escribe el token que necesita `zitadel-login` para funcionar. | `/zitadel/bootstrap/login-client.pat` |
| Login Client Machine Username/Name | **No tocar.** Identidad de cuenta de servicio interna que Zitadel crea para que la interfaz de login se autentique con ella. | `login-client` |
| Login Client PAT Expiration Date | **No lo dejes vacío.** Una fecha válida en formato RFC3339 hasta la que es válido el token del login-client. Sin esto, Zitadel se salta en silencio la escritura del fichero del token — sin ningún error en ningún sitio, `zitadel-login` simplemente nunca funciona. El valor por defecto está a ~100 años vista. | `2124-01-01T00:00:00Z` |
| Login V2 Required / Base URI | `Required` siempre es `true` en v4 (visible pero no pensado para editarse). `Base URI` es la URL pública completa donde viven las páginas de login — constrúyela con tu propio External Domain/Port/Secure de arriba. | `https://sso.example.com/ui/v2/login/` |
| OIDC Default Login/Logout URL V2 | A dónde se redirige a los clientes OIDC para iniciar/cerrar sesión. Mismo dominio que arriba, distinto sufijo. | `.../ui/v2/login/login?authRequest=`, `.../ui/v2/login/logout?post_logout_redirect=` |
| SAML Default Login URL V2 | La misma idea, para clientes SAML (p.ej. Zabbix). Solo hace falta si vas a usar SAML. | `.../ui/v2/login/login?samlRequest=` |
| Bootstrap Volume | Una carpeta compartida con `zitadel-login` (montada de solo lectura ahí) para que pueda leer el token de arriba. Ambas plantillas deben apuntar a **exactamente la misma** ruta de host. | `/mnt/user/appdata/zitadel/bootstrap` |
| Web UI / API Port | Puerto interno. Pon tu proxy inverso delante — no lo expongas directo a internet. | `8080` |
| HTTP Client Deny List (protección SSRF) | Zitadel se niega por defecto a llamar a endpoints de Target en rangos de IP privados — **esto incluye toda tu red local** (`192.168.0.0/16`). Si estás construyendo algo que llame de vuelta a tu propia red (p.ej. un webhook propio de aprobación de login), pon la variable *HTTP Client Deny List* de arriba — su valor por defecto aquí ya quita solo `192.168.0.0/16` de la lista original de Zitadel, dejando intactos el resto de rangos protegidos (loopback, link-local, endpoints de metadatos de nube). Esta variable solo surte efecto en el contenedor principal `zitadel`, y requiere recrearlo (un simple reinicio no aplica una variable de entorno nueva). | ver columna Por defecto |

#### `zitadel-login`

| Variable | Para qué sirve de verdad | Ejemplo |
|---|---|---|
| ZITADEL API URL | Cómo llega este contenedor al contenedor principal `zitadel` internamente — su IP y puerto de LAN/contenedor, **no** tu dominio público. | `http://192.168.1.50:8080` |
| Public Base Path | La ruta URL bajo la que se sirve esta interfaz de login. Debe coincidir con lo que enruta tu proxy inverso hacia aquí. Déjalo tal cual salvo que sepas por qué cambiarlo. | `/ui/v2/login` |
| Service User Token File | **No tocar.** Dónde lee este contenedor, en su interior, el token que escribió `zitadel`. Debe coincidir con la ruta del Bootstrap Volume de abajo. | `/zitadel/bootstrap/login-client.pat` |
| Custom Request Headers | Le dice a la app de login qué Host/protocolo debe creer que se le sirve, ya que está detrás de un proxy inverso que ya terminó el HTTPS. Sin esto, las cookies/redirecciones pueden romperse. | `Host:sso.example.com,X-Forwarded-Proto:https` |
| Bootstrap Volume | Debe ser **exactamente la misma ruta de host** que el Bootstrap Volume de la plantilla `zitadel` — así es como se comparte el token entre los dos contenedores. | `/mnt/user/appdata/zitadel/bootstrap` |
| Login UI Port | Puerto interno. Tu proxy inverso enruta `/ui/v2/login/*` aquí (ver abajo). | `3000` |

### ⚙️ Configuración del proxy inverso (obligatoria)

Tu proxy inverso necesita enviar dos patrones de URL distintos a dos contenedores distintos:

| Ruta | Va a |
|---|---|
| `/ui/v2/login/*` | contenedor `zitadel-login`, puerto `3000` |
| todo lo demás (`/ui/console`, `/oauth/*`, `/oidc/*`, `/api/*`...) | contenedor `zitadel`, puerto `8080` |

**Ejemplo con Nginx Proxy Manager:** en el proxy host que ya apunta a tu contenedor `zitadel`
(puerto 8080) como de costumbre, añade una **Custom Location**:
- Location: `/ui/v2/login`
- Forward Hostname/IP: la IP de tu contenedor `zitadel-login`
- Forward Port: `3000`

Sin esto, el flujo de login OIDC/SAML redirige correctamente pero acaba en un 404 justo cuando
necesita mostrar el propio formulario de inicio de sesión.

### 🩹 Solución de problemas

- **El contenedor `zitadel` se reinicia en bucle, el log dice `password authentication failed for
  user "postgres"`**: tu contenedor de Postgres se creó con un `POSTGRES_USER` personalizado (no el
  `postgres` por defecto), lo cual *sustituye* al superusuario por defecto en vez de añadir uno
  segundo. Pon ese mismo usuario personalizado en el campo *Database Admin Username* de la
  plantilla `zitadel`, en vez de `postgres`.
- **La página de login muestra un 404 en `/ui/v2/login/login`**: `zitadel-login` no está corriendo,
  no es alcanzable, o tu proxy inverso no está enrutando esa ruta hacia él — ver el apartado de
  arriba.
- **`zitadel-login` no puede leer el token / se queda unhealthy**: su ruta de Bootstrap Volume debe
  ser *exactamente* la misma ruta de host que la del contenedor `zitadel`, y `zitadel` debe haber
  completado su primer arranque (que es cuando escribe el fichero del token) antes de que arranque
  `zitadel-login`.
- **`zitadel` registra `setup completed` sin errores, pero la carpeta del Bootstrap Volume se queda
  completamente vacía (nunca aparece `login-client.pat`)**: dos causas independientes, comprueba
  ambas.
  1. **Permisos de la carpeta.** Zitadel corre internamente como UID 1000. Si Unraid creó
     automáticamente la carpeta del Bootstrap Volume en el primer arranque, probablemente quedó
     como propiedad de `root` o `nobody` sin permiso de escritura para el UID 1000 — la escritura
     del token falla por completo en silencio, sin nada registrado en ningún sitio. Arreglo: en el
     host de Unraid, `chown -R 1000:1000 <tu ruta de bootstrap>`.
  2. **Falta la fecha de expiración del PAT.** Si *Login Client PAT Expiration Date* se dejó vacío,
     Zitadel se salta por completo la creación del token, de nuevo sin ningún log. Arreglo: pon un
     valor (ver la tabla de referencia de arriba).
  En cualquiera de los dos casos, **reiniciar el contenedor por sí solo no lo arregla** — todo el
  proceso de primera instancia (crear el usuario admin, la organización, la cuenta de servicio
  login-client y su token) solo se ejecuta una vez, la primerísima vez, contra una base de datos
  genuinamente vacía. Si ya se ejecutó una vez y falló a medias (p.ej. escribió las filas en la BD
  pero no el fichero), hay que vaciar la base de datos de Postgres por completo y volver a arrancar
  `zitadel` para que lo reintente desde cero — arreglar antes los permisos/la variable y solo
  reiniciar no cambia nada.
- **Crear un Target de Actions V2 falla con `[invalid_argument] Errors.Target.DeniedURL`**: por
  defecto Zitadel se niega a llamar a endpoints webhook en rangos de IP privados, como protección
  SSRF — esto incluye toda tu red local (`192.168.0.0/16`). Si estás construyendo algo que llame de
  vuelta a tu propia red (p.ej. un webhook propio de aprobación de login), pon la variable *HTTP
  Client Deny List* de arriba — su valor por defecto aquí ya quita solo `192.168.0.0/16` de la
  lista original de Zitadel, dejando intactos el resto de rangos protegidos (loopback, link-local,
  endpoints de metadatos de nube). Esta variable solo surte efecto en el contenedor principal
  `zitadel`, y requiere recrearlo (un simple reinicio no aplica una variable de entorno nueva).

### 🔐 `oauth2-proxy` — poner ese mismo login delante de cualquier otra cosa

La tercera plantilla de este repositorio no forma parte de ZITADEL: es
[**oauth2-proxy**](https://oauth2-proxy.github.io/oauth2-proxy/), la pieza que permite reutilizar el
proveedor de identidad que acabas de desplegar para proteger aplicaciones web que **no tienen login
propio** — la página de un router, un panel de monitorización, un administrador antiguo. Tu proxy
inverso le pregunta a oauth2-proxy, en cada petición, si el visitante ha iniciado sesión; si no, lo
manda a tu proveedor y solo vuelve una vez autenticado.

En Community Applications ya había una plantilla de oauth2-proxy, y funciona, pero **no expone ni un
solo ajuste**: todo hay que escribirlo a mano en un fichero de configuración. Esta expone cada
opción como un campo, explicando qué hace y por qué importa.

**Un contenedor, muchas webs.** Deja *Redirect URL* como una ruta (`/oauth2/callback`) en vez de una
dirección completa: oauth2-proxy deduce entonces la URL absoluta a partir del dominio de cada
petición, así que el mismo contenedor vale para cuantos dominios quieras. Pon un punto delante en
*Cookie domains* (`.ejemplo.com`) y un solo inicio de sesión abre todas las webs protegidas.
*(No uses el flag `--relative-redirect-url` para esto: manda un `redirect_uri` literalmente
relativo, que los proveedores que cumplen el estándar rechazan.)*

**Nginx Proxy Manager**, que no tiene ningún campo de `auth_request` en su interfaz, lo admite de
todos modos: lo que escribas en la pestaña *Advanced* de un host se inserta a nivel de servidor, y
NPM se salta su propio `location /` en cuanto tu configuración declara uno. Con esto basta, en cada
web protegida:

```nginx
location /oauth2/ {
    proxy_pass         http://IP_DE_OAUTH2_PROXY:4180;
    proxy_set_header   Host              $host;
    proxy_set_header   X-Real-IP         $remote_addr;
    proxy_set_header   X-Forwarded-For   $proxy_add_x_forwarded_for;
    proxy_set_header   X-Forwarded-Proto $x_forwarded_proto;
}

location = /oauth2/auth {
    internal;
    proxy_pass              http://IP_DE_OAUTH2_PROXY:4180;
    proxy_set_header        Host              $host;
    proxy_set_header        X-Real-IP         $remote_addr;
    proxy_set_header        X-Forwarded-For   $proxy_add_x_forwarded_for;
    proxy_set_header        X-Forwarded-Proto $x_forwarded_proto;
    proxy_set_header        Content-Length    "";
    proxy_pass_request_body off;
}

location @oauth2_signin {
    return 302 https://$host/oauth2/start?rd=$request_uri;
}

location / {
    auth_request /oauth2/auth;
    error_page 401 = @oauth2_signin;

    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection $http_connection;
    proxy_http_version 1.1;

    include conf.d/include/proxy.conf;
}
```

Registra `https://esa-web/oauth2/callback` como redirect URI en tu proveedor para **cada** web que
protejas, y añade el correo del visitante al fichero de la lista blanca. Dos cosas que conviene
saber antes de confiarle nada:

- **Quien entra, entra en todo** lo que haya detrás de esa misma instancia. Para otro público
  (familia, invitados) levanta un segundo contenedor con su propio nombre de cookie, su propia
  aplicación en el proveedor y su propia lista blanca.
- **No lo pongas delante de tráfico de máquinas** — sondas de monitorización, apps móviles,
  webhooks, APIs. Es una puerta para navegadores; cualquier cosa no interactiva se quedará fuera en
  la redirección.

### 💙 Apoya el proyecto

Sin el apoyo de la comunidad estos proyectos no serían posibles. Si te ha resultado útil, puedes
apoyarlo vía [GitHub Sponsors](https://github.com/sponsors/Nebur692),
[Ko-fi](https://ko-fi.com/nebur69265723) o [PayPal](https://paypal.me/0SkillS) — cualquier
aportación ayuda a seguir manteniéndolo.

### ⚠️ Aviso legal

No afiliado, respaldado ni asociado con ZITADEL ni ZITADEL GmbH. "ZITADEL" es una marca registrada
de su respectivo propietario. No afiliado con jcesclapez/unraid-templates más allá de reconocer que
es la razón por la que existe este repositorio.

### 📜 Licencia

[MIT](LICENSE)
