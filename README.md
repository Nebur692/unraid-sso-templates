<div align="center">

# unraid-zitadel-templates

*Working Unraid Community Applications templates for ZITADEL v4 — core + the login UI it actually needs*

![Release](https://img.shields.io/github/v/release/Nebur692/unraid-zitadel-templates?label=release&color=blue)
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

### 📦 Installation

1. **Add this repository** in **Docker → Add Container → scroll down → Template repositories**:
   ```
   https://raw.githubusercontent.com/Nebur692/unraid-zitadel-templates/main/zitadel/zitadel.xml
   https://raw.githubusercontent.com/Nebur692/unraid-zitadel-templates/main/zitadel-login/zitadel-login.xml
   ```
2. You need an external **PostgreSQL 14+** container first (not included — any standard Postgres
   template works). Leave `POSTGRES_USER` at its default (`postgres`) unless you have a specific
   reason to change it — see the warning in the `zitadel` template's *Database Admin Username*
   field if you do.
3. **Add Container → zitadel** from the template dropdown. Fill in at minimum: Master Key (32
   random characters, generate once and keep it — you cannot change it later without losing
   encrypted data), External Domain, Database connection details, and the four
   `*_LOGINURLV2`/`LOGINV2_BASEURI` fields (all derived from your External Domain/Port — the
   description on each field shows the exact format).
4. **Add Container → zitadel-login** from the template dropdown. Fill in `ZITADEL_API_URL` (the
   `zitadel` container's internal IP and port, e.g. `http://192.168.1.50:8080`) and
   `CUSTOM_REQUEST_HEADERS` (your domain). Point its **Bootstrap Volume** at the exact same host
   path you used for the `zitadel` container — this is how it reads the token the core container
   generates on its first start.
5. Start `zitadel` **first**, confirm it logs `setup completed` with no errors, **then** start
   `zitadel-login`.

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

### 📦 Instalación

1. **Añade este repositorio** en **Docker → Add Container → baja hasta Template repositories**:
   ```
   https://raw.githubusercontent.com/Nebur692/unraid-zitadel-templates/main/zitadel/zitadel.xml
   https://raw.githubusercontent.com/Nebur692/unraid-zitadel-templates/main/zitadel-login/zitadel-login.xml
   ```
2. Necesitas primero un contenedor externo de **PostgreSQL 14+** (no incluido — cualquier plantilla
   estándar de Postgres vale). Deja `POSTGRES_USER` en su valor por defecto (`postgres`) salvo que
   tengas una razón concreta para cambiarlo — mira el aviso en el campo *Database Admin Username*
   de la plantilla `zitadel` si lo haces.
3. **Add Container → zitadel** desde el desplegable de plantillas. Rellena como mínimo: Master Key
   (32 caracteres aleatorios, genéralo una vez y guárdalo — no se puede cambiar después sin perder
   datos cifrados), External Domain, los datos de conexión a la base de datos, y los cuatro campos
   `*_LOGINURLV2`/`LOGINV2_BASEURI` (todos derivados de tu External Domain/Port — la descripción de
   cada campo muestra el formato exacto).
4. **Add Container → zitadel-login** desde el desplegable de plantillas. Rellena `ZITADEL_API_URL`
   (la IP y puerto internos del contenedor `zitadel`, p.ej. `http://192.168.1.50:8080`) y
   `CUSTOM_REQUEST_HEADERS` (tu dominio). Pon su **Bootstrap Volume** apuntando exactamente a la
   misma ruta de host que usaste en el contenedor `zitadel` — así es como lee el token que el
   contenedor principal genera en su primer arranque.
5. Arranca **primero** `zitadel`, confirma que registra `setup completed` sin errores, **luego**
   arranca `zitadel-login`.

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
