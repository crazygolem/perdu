# Deploying perdu

Perdu is one binary, packaged as a container image.

It needs read access to the Docker API, to find the services your labels describe, and is meant to run behind a reverse proxy, which authenticates your users and tells perdu who they are.
[Non-standard setups](#non-standard-setups) gives hints for running it without one, or outside a container.

By default, perdu listens on all interfaces on port 8080, configurable via `--listen`.

See [flags and environment variables](flags.md) for detailed descriptions of the available options and required flags.
Run `perdu -h` for a summary.

## The Docker image

Pull the image from GitHub's container registry:

```sh
docker pull ghcr.io/crazygolem/perdu:0.1
```

Each release's image is tagged with its version, such as `0.1.0`; a final release's also moves its minor line's tag, `0.1`, and `latest`, which a pre-release never moves.
Until perdu's 1.0, a minor release may break your setup: follow a minor line's tag, and read the release notes before moving to the next.
The image is built for `linux/amd64` and `linux/arm64`.

The image runs the `perdu` binary as its entry point, as an unprivileged user.
The image also holds the CA certificates that fetching `di:` icons needs, and the licences of what it ships, in `/licenses/`.

## Compose stack example

The following snippet shows a stack where perdu reaches the Docker API through a socket proxy (see [Access to the Docker API](#access-to-the-docker-api)), with one labelled service:

```yaml
services:
  perdu:
    image: ghcr.io/crazygolem/perdu:0.1
    command:
      - --default-locale=en
      - '--audience-expr=headers["remote-groups"] | split(",")'
      - '--data=name=headers["remote-name"]'
    environment:
      DOCKER_HOST: tcp://socket-proxy:2375
    networks:
      - default
      - docker-api
    depends_on:
      - socket-proxy
    restart: unless-stopped

  socket-proxy:
    image: tecnativa/docker-socket-proxy:v0.5.0
    environment:
      # The container listing, which opens every GET under /containers/
      # (see the note below); the event stream and _ping are on by
      # default, and the methods that write are off.
      CONTAINERS: 1
    networks:
      - docker-api
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    restart: unless-stopped

  jellyfin:
    image: jellyfin/jellyfin
    labels:
      perdu.enable: "true"
      perdu.name: Jellyfin
      perdu.description: Films and series
      perdu.url: https://jellyfin.example.com
      perdu.icon: di:jellyfin
      perdu.category: Media

networks:
  # The default network, with access to the internet; used by default by
  # services that don't declare a network.
  default:
  # Perdu's and the socket proxy's alone, with no way out.
  docker-api:
    internal: true
```

Your reverse proxy joins the `default` network and sends perdu's traffic to `perdu:8080`, adding the `Remote-Groups` and `Remote-Name` headers the two expressions read ([The reverse proxy](#the-reverse-proxy)).

[Labels](labels.md) describes every label, and [Per-viewer data](rules.md) the expressions.

> [!TIP]
> In `command:`, write each flag as one item with its value after `=`, and quote every item that holds an expression.
> Unquoted, YAML reads `: ` as the start of a mapping and a `#` after a space as the start of a comment: a ternary such as `a ? b : c` makes Compose reject the file, and `or # == "oncall"` cuts the expression short, which perdu then refuses as not compiling.

> [!NOTE]
> This is a **simplified example** to get you started quickly, in a trusted environment.
> A production setup should be hardened; read on to find out how.

## Access to the Docker API

Perdu reads the Docker API and never changes anything through it.
These are the calls it makes, so that you can open a proxy to them alone, each path except the ping's under the API version, as in `/v1.47/containers/json`:

| Call                   | What it reads                                                                                                                                                                                                                   |
| ---------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `HEAD /_ping`          | the API version the daemon speaks, before the first call, unless `DOCKER_API_VERSION` gives one                                                                                                                                 |
| `GET /containers/json` | the containers, stopped ones included, filtered by label: those carrying `<prefix>.enable`, and those sharing the identity of a service whose containers are all stopped ([How perdu works](how-it-works.md#what-a-build-does)) |
| `GET /events`          | the event stream: the containers carrying `<prefix>.enable` being created, updated or removed                                                                                                                                   |

Each call has 5 seconds to answer, the event stream's headers included; a call that doesn't fails like any other failure, and perdu retries it ([Starting](#starting)).

**Prefer a socket proxy to mounting the socket.**
Mounting `/var/run/docker.sock` into a container is root-equivalent on the host: whatever can call that socket can start a container with any privilege it likes.
A code-execution bug in perdu — a parsing bug, a compromised dependency — would then be a compromise of the host, not just of perdu.
This is guidance perdu's own code can't enforce: it uses whatever access you give it.

A proxy that filters by method and path can allow the calls above selectively: [`wollomatic/socket-proxy`](https://github.com/wollomatic/socket-proxy) and [CetusGuard](https://github.com/hectorm/cetusguard) each let a request through only where its method is one you allow and its path matches a regular expression you give for it.
The example stack uses `tecnativa/docker-socket-proxy` as an example, being simpler to set up, though it allows more than strictly needed.

> [!NOTE]
> `tecnativa/docker-socket-proxy` allows the container listing through its `CONTAINERS` switch, which allows every `GET` under `/containers/` with it: a container's details, its environment included, its logs and its files.
> On Docker Engine, one of those allows attaching to a container's stdin using a websocket, so limiting the containers endpoint to the `GET` method can still allow write operations.

**A socket proxy answers whatever reaches its port**, unauthenticated and in plaintext.
On a network shared with the rest of the stack, any of its containers could read every container's environment, logs and files through it, and more.
The example stack puts the proxy on a network only perdu shares, declared `internal: true` so that the proxy reaches nothing outside either.

Reached over TCP, as in the example stack, the socket proxy needs nothing more from perdu's container, which keeps running as its unprivileged user.
If you do mount the socket, that user needs access to it: add the socket's group with compose's `group_add` (on most hosts the `docker` group, by its number), or run the container as a user who has access with `user:`.
Discovery failing with a permission error on the socket at startup is this.

**The connection is set by Docker's own variables**, with the meanings Docker's client gives them: `DOCKER_HOST`, `DOCKER_TLS_VERIFY`, `DOCKER_CERT_PATH`, `DOCKER_API_VERSION` and the other `DOCKER_*` variables Docker's client reads.
Perdu has no flag for any of this.

**`DOCKER_TLS_VERIFY` is what checks the daemon's certificate**, against the `ca.pem` in `DOCKER_CERT_PATH` alone (the system's CA certificates are not used).
Without it, `DOCKER_CERT_PATH` encrypts the connection but accepts any server, its `ca.pem` included, as it does in Docker's client.

Perdu refuses to start, naming the variable, where Docker's client would quietly do less than you asked:

* `DOCKER_TLS_VERIFY` set without `DOCKER_CERT_PATH`, which Docker's client ignores;
* a `DOCKER_HOST` with the `https` scheme and no `DOCKER_CERT_PATH`, or with a scheme other than `unix`, `tcp`, `http` and `https`, such as `ssh`, which Docker's client would speak as plain HTTP;
* `DOCKER_TLS` set, which asks the `docker` command for TLS but which Docker's client doesn't read.

It also refuses a `DOCKER_HOST` or a `DOCKER_API_VERSION` that doesn't parse, and certificate files it can't read.

The daemon can be Docker Engine, or podman through its Docker-compatible API.

The Docker API connection **never goes through a proxy**, whatever `HTTPS_PROXY` and the other proxy variables say ([Fetching icons through a proxy](#fetching-icons-through-a-proxy)): Docker can set those variables in every container it creates, where they would send the calls meant for a socket proxy on the compose network to a proxy that can't reach it.
A daemon reachable only through a forward proxy is not supported.

## The reverse proxy

Perdu does not perform authentication nor authorization.

It expects a reverse proxy in front of it that has already authenticated the user and forwards who they are in request headers, which your expressions read.
Avoid forwarding user attributes via the query: whoever wrote the link wrote the query, including another site ([Per-viewer data](rules.md#ways-a-rule-can-surprise-its-author)).

Perdu does not enforce specific header names for user data; your proxy can forward e.g. `Remote-Groups` or `X-Forwarded-Groups`; you are in charge of writing expressions that use the headers you set ([Per-viewer data](rules.md)).

Perdu treats those headers as trustworthy for presentation only: they decide what the page shows a viewer, never what the viewer may reach.

**Perdu's port should only be reachable through your proxy.**
Perdu ends a request that stops making progress: one not read whole within 5 seconds, and a response that accepts nothing for 30 seconds.
Your proxy is better equipped to stop anything subtler, such as a client trickling bytes inside those limits or opening many connections.

> [!TIP]
> Perdu can also work without a reverse proxy in front of it, e.g. for a simple stack where you don't care about the per-user dynamic page features, in an environment where perdu is not exposed: see [Without a reverse proxy](#without-a-reverse-proxy).

### The client's address

The client's address is logged on every line about that request.

When the request arrives from a trusted proxy (see [`--trusted-proxies`](flags.md#--trusted-proxies)), the client's address is read from the `X-Forwarded-For` header:
Perdu walks `X-Forwarded-For` from the right and takes the first address that isn't a trusted proxy's. When the header runs out of entries, or an entry isn't an address, it takes the last trusted address it reached.

Without `--trusted-proxies`, every request coming from your reverse proxy logs your proxy's address as source instead of the client's actual address.

**Trust your proxy's own address rather than its network's range.**
Every address an entry covers can choose the client address logged for its own requests, by sending `X-Forwarded-For`, and a forged address in a ban list built from the log lands on someone else.
A container's address changes when it is recreated, which makes the range of a compose network the natural entry, but that trusts every container on the network: fix your proxy's address in the stack instead, with compose's `ipv4_address`, or trust a network only your proxy and perdu share.

**Your proxy must set or append `X-Forwarded-For` with bare addresses**, such as `203.0.113.7` or `2001:db8::7`.
An entry carrying a port, such as `203.0.113.7:51234`, isn't an address, so it ends the walk there.
An address in a zone, such as `fe80::1%eth0`, is never a trusted proxy, so your proxy has to reach perdu by an address its network assigns rather than a link-local one.

> [!NOTE]
> The `Forwarded` and `X-Real-IP` headers are not supported.

Perdu reads the client's address only to log it, and logs it normalized: an IPv4-mapped IPv6 address is the IPv4 address, and an IPv6 address takes its shortest standard spelling, so a ban list built from the log sees one spelling per address.

### The client's scheme

Perdu sets the `Secure` flag on its cookies when the viewer reached your proxy over HTTPS, which is determined via the `X-Forwarded-Proto` header (its first element when proxies joined several, compared case-insensitively).

If your proxy doesn't send that header reliably, set `--cookie-secure=true` (or `false`) and perdu stops looking at the header.
`false` in production is usually a mistake.

A client able to forge the header changes only whether its own cookies are marked `Secure`.

### Paths

Perdu serves every path under its base path (`{base}` in the table below, and implied in the rest of the documentation), determined via `--base-path` and the root `/` by default.
**Your proxy must forward the full path unstripped.**

| Path                       | What it answers                                                                                                                                          |
| -------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `{base}`                   | a redirect to `{base}/`, where the base path isn't the root                                                                                              |
| `{base}/`                  | a redirect to the page in the viewer's language ([Locales](locales.md#which-page-a-viewer-lands-on))                                                     |
| `{base}/l/<locale>/`       | the page in that language, or for a language with no page, a redirect to the nearest one, or an error page ([Locales](locales.md#a-locale-with-no-page)) |
| `{base}/l/<locale>`        | a redirect to `{base}/l/<locale>/`                                                                                                                       |
| `{base}/set-lang/<locale>` | stores the viewer's choice of language, then redirects to `{base}/l/<locale>/`                                                                           |
| `{base}/assets/…`          | the stylesheet, the scripts, the favicon and the icons the pages use                                                                                     |

Each of these answers `GET` and `HEAD`.
Any other method gets a `405`; every other path gets a `404`.
A path written with an empty, `.` or `..` segment, such as `/l/en/../en/`, is redirected to its clean form first.

### Caching

Every response perdu sends carries `Cache-Control: no-cache`, so a cache in front of perdu may keep a copy but asks perdu again before using it and gets a cheap `304` when nothing changed: pages and assets carry an `ETag` computed from their content.

Pages and redirects that depend on the viewer's cookies or language carry `private, no-cache` instead, telling shared caches to not store those responses.

Until the first page has been built, every request gets a `503` with `no-store` ([Starting](#starting)).

### Content Security Policy

Perdu sets the `Content-Security-Policy` header **only on icon responses**.

> [!WARNING]
> **Leave perdu's own policy in place on icon responses.**
> Perdu sends `Content-Security-Policy: sandbox` on every response under `/assets/icons/`, and a policy your proxy *sets* there replaces it: with the inline scripts perdu's pages need, an SVG icon's script would then run in perdu's origin, in a browser that shows the icon rather than downloading it ([Icons](icons.md#safety)).
>
> Have your proxy add its header beside perdu's rather than replace it, leave that path out, or include `sandbox` in the policy it sets there.

If your proxy adds one to perdu's paths, **it has to allow inline scripts, inline styles and `data:` images**: perdu embeds its scripts, structural stylesheets and fallback icons inside each page.
For example:

```
Content-Security-Policy: default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline'; img-src 'self' data:
```

Additionally, add to `img-src` the hosts of any icon your labels provide as an `http(s)` URL ([Icons](icons.md)).

> [!NOTE]
> **The stricter forms cannot be used**: as detailed below, a hash can't be used for all of perdu's scripts, and perdu doesn't generate pages per-request so nonces cannot be injected.
>
> As a result, a policy that works with perdu cannot be considered a second line of defence: if a bug in perdu's escaping lets markup through, the policy would not stop the script it carried.

Each of the following costs its own part of the page when a policy refuses it ([The template](template.md#under-a-content-security-policy)):

* **Inline scripts** carry everything the page does with per-viewer data, and its presentational touches.
  Without them, the page is the one a browser with JavaScript disabled shows, with nothing customized for the viewer.
* **Inline styles** carry the page's structural rules: some the scripts pair with, and others standing on their own.
  Without them, filtering fails open, parts of the page show where they aren't meant to, and some elements aren't drawn as intended: a failed icon shows what the browser draws for one, the language dropdown opens in its default place, a plate's letters follow a viewer's text-spacing override, and links under forced colours show in the text colour in Firefox.
* **`data:` images** carry the fallback icons, which are missing without them.

Allowing inline scripts cannot be substituted with allowing them by their hash: one of the scripts carries the page's [environment identity](rules.md#the-environment-identity), which can change with any build.

**Allowing inline scripts and styles for perdu is not inherently unsafe.**
What leaving out `'unsafe-inline'` protects against is scripts and styles that an attacker manages to inject into a page: with inline content allowed, the browser can no longer tell them apart from the page's own.
That protection matters on a page that writes attacker-controlled input into its markup, which perdu does not do:

* A page is built before any request reaches it, and every viewer receives the same page (for a given language): nothing from a request, whether its path, its query string or its headers, is written into it.
* What a page holds comes from your labels, and perdu escapes each value where it renders it, whoever wrote the label, an image's own labels included.
  Text stays text, and a [`url`](labels.md#url) keeps the escaping that turns `javascript:` (and every other scheme perdu doesn't accept) into an inert link.
* Per-viewer data, which your expressions read from your proxy's headers, reaches the page only in cookies, and the page's scripts write it into the page as text, never as markup.

Allowing `data:` images opens nothing either: browsers run no script in an image (including for SVG images), and perdu shows every service's and action's icon as an image ([Icons](icons.md#safety)).
The rest of the policy still holds: under `default-src 'self'` the page loads no script, stylesheet, font or frame from another origin and no script on it can fetch from one, and `img-src` keeps its images to perdu's host, `data:` and the hosts you add.

### Embedded in another site

Browsers send and store perdu's cookies only when the page embedding perdu, if any, is on the same site, such as another host name of the same domain.
Embedded in a frame under another site, perdu's page receives none of its per-viewer data, so nothing on it is customized for the viewer, and a viewer's choice of language isn't kept.

## Sharing a host name with other applications

Perdu can share a host name with other applications (including other perdu instances), provided none of them is served at or under perdu's base path.

Two perdu instances on one host name need different base paths and different `--namespace`s, since the namespace names their cookies.

Applications on the same host name **share perdu's origin and its cookie jar**.
Since perdu's cookies aren't `HttpOnly`, the other applications' scripts can read them, and any application on the host name (or on another host name of the same domain, through a cookie set for the whole domain) can write cookies under perdu's names.
Perdu uses cookies for presentational purposes, not for access control, but you are sharing a cookie jar, not just an origin.

> [!WARNING]
> Don't build another application on reading or writing perdu's cookies: their format is internal and may change without notice.

## Fetching icons through a proxy

Perdu fetches `di:` icons from jsDelivr's CDN ([Icons](icons.md#di-dashboard-icons)), and those fetches honour the standard proxy variables, `HTTPS_PROXY`, `HTTP_PROXY` and `NO_PROXY`, in upper or lower case.

Nothing else perdu sends goes through them, including the Docker API connection.

## Starting

On startup perdu checks every flag: the first one it can't use stops it at once with an `ERROR` and a non-zero exit status.
It then listens on its addresses: an address the system refuses also stops it at once.

Perdu then waits for two things, side by side, for at most **60 seconds**:

* the icons directory, if `--icons-dir` is set, to exist and be a readable directory;
* the Docker event stream to connect.

Once both are there, the first build lists the containers and renders the pages, and perdu logs `startup complete` once they are served.
Until then every request gets a `503`.

A socket proxy, a daemon or a network mount that comes up a few seconds after perdu is the ordinary case in a compose stack, and is retried within the window.
If the window ends before the first listing has succeeded, perdu exits with a non-zero status, so that a wrong `DOCKER_HOST` or mount path ends in a container that visibly stopped rather than one that looks up and never serves.
A first build that fails while rendering also stops perdu, since there is no page to serve.

When perdu starts as part of a compose stack, its first page can show part of the stack, and the builds that follow the other containers' events complete it within seconds.

Once perdu has served a page, it keeps serving: Docker becoming unreachable, a listing failing, or the event stream dropping leaves the last page served while perdu retries in the background, and logs that it does ([Reading the logs](logs.md)).

## Refreshing

Sending perdu `SIGHUP` asks it to refresh:

```sh
docker compose kill -s HUP perdu
```

A refresh reloads every icon the page uses, reopens the Docker event stream, and builds the page again from a fresh listing ([How perdu works](how-it-works.md#refreshes)): it is how a change to the files in `--icons-dir` takes effect.
Use it after changing your icons, after a `di:` icon changed upstream, or when the page seems to have stopped following your containers.

The signal has to reach perdu's own process.
The image runs perdu directly, so it does; an image of your own whose entry point is a shell script has to forward the signal or `exec` perdu, or run an init such as `tini`, or the refresh silently never happens.

Flags are read only at startup: changing one takes a restart.

## When a change reaches the page

| You change                                                               | It shows                                                                                                                |
| ------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------- |
| A container's labels, by recreating it                                   | on the build its create, update or removal event triggers, a second or so later                                         |
| Which of a service's containers run, where they declare different labels | on the next build, since starting or stopping a container triggers none ([Labels](labels.md#service-identity))          |
| A flag                                                                   | after a restart                                                                                                         |
| A file in `--icons-dir`                                                  | after a refresh, or for an icon no page has used yet, on the build after the one that first uses it ([Icons](icons.md)) |
| A `di:` icon upstream                                                    | after a refresh                                                                                                         |

[How perdu works](how-it-works.md) explains why each change shows when it does.

## Stopping

On `SIGTERM` or `SIGINT`, perdu stops accepting connections, gives the requests in progress up to 5 seconds to finish, and exits with status 0.

Perdu exits with a non-zero status when it can't start, when the startup window ends, when a listener stops accepting connections, and when one of its own components fails in a way it can't recover from.

## Non-standard setups

This guide runs perdu from its published image, behind a reverse proxy.
Perdu works without either, with a few things to watch for.

### Without a reverse proxy

Perdu's defaults need no proxy, but without one:

* **Nothing authenticates the viewer**: the headers your rules read are whatever the viewer's browser sends, so per-viewer data built on them means nothing, though a forged value changes only what its sender sees.
  Leave `--audience-expr` and `--data` unset, unless their rules read something else.
* **Perdu serves plain HTTP only**, which suits a trusted network, never the internet.
* **Leave `--cookie-secure` at `auto`**, which marks no cookie `Secure` without a proxy's `X-Forwarded-Proto`: `true` would have browsers refuse the cookies over plain HTTP.
* **Leave `--trusted-proxies` unset**, so that the log names each viewer's own address.
* **Publish perdu's port**, with compose's `ports:`, for viewers to reach it.
* **Nothing bounds a client trickling bytes or opening many connections**: perdu ends only a request that stops making progress ([The reverse proxy](#the-reverse-proxy)).

### Outside a container

Perdu runs the same as a plain binary, extracted from the image or [built from its sources](#built-from-the-sources), but what the image arranged is then yours:

* **Listening**: perdu listens on every address of the machine by default; for local uses or behind a proxy on the same machine, `--listen localhost:8080` keeps it off the network.
* **The Docker API**: without `DOCKER_HOST`, perdu uses the daemon's socket, `/var/run/docker.sock`, as Docker's client does, so the user running perdu needs access to it, which is root-equivalent ([Access to the Docker API](#access-to-the-docker-api)); `DOCKER_HOST` can point it at a socket proxy instead.
* **Refreshing**: `SIGHUP` goes to perdu's own process, e.g. `kill -HUP <pid>` ([Refreshing](#refreshing)).
* **CA certificates**, for `di:` icons, are the system's.

### Built from the sources

> [!NOTE]
> Perdu's sources aren't public yet: building perdu requires a copy of its repository.

Build the binary from the sources:

```sh
go build ./cmd/perdu
```

Build the image from the repository's `Dockerfile`:

```sh
docker build -t perdu .
```
