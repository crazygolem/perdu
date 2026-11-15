# Icons

Services and actions show an icon, configured with `<prefix>.icon` and `<prefix>.actions.<key>.icon`.
Without one, the page draws a default: for services their initials on a coloured plate, and for actions the default action icon ([How the page draws icons](#how-the-page-draws-icons)).

## The forms an icon takes

| Form                               | Example                                       | What perdu does                                                                              |
| ---------------------------------- | --------------------------------------------- | -------------------------------------------------------------------------------------------- |
| a library icon, `<library>:<name>` | `fa:solid-key`, `di:jellyfin`, `logos:backup` | resolves it from that library ([Libraries](#libraries)) and serves it under `/assets/icons/` |
| an `http` or `https` URL           | `https://jellyfin.example.com/favicon.ico`    | nothing: the viewer's browser fetches it                                                     |
| a root-relative path               | `/plex/favicon.ico`                           | nothing: the viewer's browser fetches it from perdu's host                                   |
| a `data:` URI                      | `data:image/png;base64,iVBORw0…`              | decodes it, and serves it under `/assets/icons/` ([`data:` icons](#data-icons))              |

* **A library icon**'s library and name must each match `[a-zA-Z0-9_-]+`, and are case sensitive: `fa` and `FA` are two libraries.
* **An `http` or `https` URL** must name a host.
  Perdu never fetches it, and never knows whether it loads; useful to show favicons from your stack's own services.
* **A root-relative path** begins with `/`, for a service served on a path under perdu's own host.
* A value starting with `http:`, `https:` or `data:`, in any case, is always that URI form, so no library can be named `http`, `https` or `data`.
* `http`, `https` and root-relative icons are parsed as strictly as a [`url`](labels.md#url).

**Perdu checks the icons it serves**, whatever the source; `http`, `https` and root-relative icons never pass through perdu, and go unchecked.
Checked icons must be **at most 256 KiB**, and declare one of the accepted formats by their extension or media type, tried by priority in your icons directory ([Your own icons](#your-own-icons)):

| Format | Extension       | Media type      | Priority |
| ------ | --------------- | --------------- | -------- |
| SVG    | `.svg`          | `image/svg+xml` | 1        |
| PNG    | `.png`          | `image/png`     | 2        |
| JPEG   | `.jpg`, `.jpeg` | `image/jpeg`    | 3        |
| WebP   | `.webp`         | `image/webp`    | 4        |
| GIF    | `.gif`          | `image/gif`     | 5        |

A value that doesn't match any of the forms above or that doesn't pass the checks is unusable: the page shows its default, with a `WARN`.
An empty value is no icon, quietly.

> [!NOTE]
> Perdu's checks are surface-level, not a complete validation; perdu does not parse the icons' bytes to ensure that they match their declared format, let alone follow its specification.

## `data:` icons

A `data:` icon has the form `data:<media type>[;base64],<data>`: media type parameters written as `name=value`, and the data in base64 after `;base64`, or percent-encoded otherwise.

Perdu decodes it, checks its type and its size, and serves the bytes from `/assets/icons/~data/`, so identical icons on several services are fetched once.

> [!NOTE]
> **Perdu decodes a `data:` URI more strictly than a browser**, so one that shows in your browser can still be refused here, for example:
>
> | Refused                                                                | Write instead                                               |
> | ---------------------------------------------------------------------- | ----------------------------------------------------------- |
> | a parameter that isn't `name=value`: `data:image/svg+xml;utf8,<svg …>` | `data:image/svg+xml;charset=utf-8,<svg …>`, or no parameter |
> | base64 without its padding, the `=` it ends with                       | the padding, to a multiple of four characters               |
> | base64 in its URL-safe alphabet, with `-` and `_`                      | the standard alphabet, with `+` and `/`                     |
> | base64 with percent-escapes, such as `%2B`                             | the characters themselves                                   |
> | no media type: `data:;base64,…` or `data:,…`                           | the image's media type                                      |
> | a `%` that starts no escape in percent-encoded data                    | `%25`                                                       |

## Libraries

**Perdu ships no icon library**: a library is a directory of files you mount ([Your own icons](#your-own-icons)), a configured remote perdu fetches its icons from ([Remotes](#remotes)), or both.
Its name is the one you give it, and `fa` or `di` is no more special than `logos`: a label naming `di:jellyfin` resolves only where you configured a `di` library, and shows whatever that library holds as `jellyfin`.
With neither [`--icons-dir`](flags.md#--icons-dir) nor [`--icons-remote`](flags.md#--icons-remote) set, no library icon resolves, and perdu contacts no icon source.

> [!NOTE]
> **Until a library is configured, its icons show as the default plates and action icons**: their labels don't resolve until a remote or a directory provides each library.
> [Collections](#collections) gives the remote configuration for a few well-known icon libraries.

**After a restart, the first built pages have no library icons**, which perdu has to read or fetch again ([How perdu works](how-it-works.md#the-resource-cache)).
They arrive on the build that follows a moment later: reload the page to see the icons.

### Remotes

A remote tells perdu where to fetch a library's icons, as URLs holding the icon's name:

```
--icons-remote 'logos:https://icons.example.com/{name}.svg'
```

Here `logos:nas` is fetched from `https://icons.example.com/nas.svg`.

Its full form is `<library>:[<pattern>] <url> [<modifier>…] [<url> [<modifier>…]]…`, one remote per library ([`--icons-remote`](flags.md#--icons-remote)):

* **A URL** is an `http` or `https` URL, with placeholders in its path or its query: `{name}` is the icon's name, or, with a [pattern](#patterns), the pattern's groups stand in its place.
  A literal `{` or `}` must be written percent-encoded, `%7B` and `%7D`.
* **Several URLs** are tried in order, moving on to the next only when one has no usable file: a `404`, another status a modifier maps to `not-found`, or a file over 256 KiB.
* **A format list**, `{format:svg,png,webp}`, tries one format after another, in the list's order, and a `{format}` elsewhere in the URL repeats the one being tried: `https://icons.example.com/{format:svg,png}/{name}.{format}` tries `…/svg/nas.svg`, then `…/png/nas.png`.
  The file must then be served with that format's media type.
  Each URL of a remote can have its own format list.
* **A URL without a list** takes the format the response declares, whichever [accepted format](#the-forms-an-icon-takes) it is, and never reads the URL's extension: `https://cdn.simpleicons.org/{name}` serves SVG, and `https://icons.example.com/{name}.svg` could serve PNG.
* **Modifiers** after a URL adapt how some response statuses from that URL are handled ([Modifiers](#modifiers)).

Whether a remote follows its upstream or is pinned is a property of the remote; some CDNs let you select one or the other via the URL: with jsDelivr, `fontawesome-free@7` follows Font Awesome's version 7, while `twemoji@14.0.2` is pinned.
jsDelivr caches a file under a version that follows upstream, such as `@7`, for up to 7 days, so a new release can take that long to reach perdu.

An image's labels can make perdu request icons only from the remotes' hosts you configured, under names the image chose ([Labels an image declares](labels.md#labels-an-image-declares)).

#### Patterns

A pattern, before the first URL, splits or restricts a name: a [regular expression in Go's syntax](https://pkg.go.dev/regexp/syntax), matched against the whole name, whose named groups, `(?<group>…)`, are the URLs' placeholders:

```
--icons-remote 'fa:(?<style>solid|regular|brands)-(?<name>.+) https://cdn.jsdelivr.net/npm/@fortawesome/fontawesome-free@7/svgs/{style}/{name}.svg'
```

Here `fa:solid-key` is fetched from `…/svgs/solid/key.svg`.

* **A name the pattern doesn't match is refused**, and so is one leaving a group empty, without a request: here `fa:light-key` is refused, unless your icons directory holds it, which perdu reads first ([Your own icons](#your-own-icons)).
* **With a pattern, `{name}` is a group like any other**, and exists only where the pattern names a group `name`.
* **Two groups may share a name**, as alternatives: `(?<style>solid|regular)-(?<name>.+)|(?<name>.+)-(?<style>solid|regular)` takes `solid-key` and `key-solid` alike, and both reach one file.
  Each alternative has to give a value to every name the pattern's groups use, since a pattern can't supply a default.
* **A pattern can't widen what a label reaches**: a name holds only `[a-zA-Z0-9_-]`, whatever the pattern says, so a group never holds a `/` or a `.`.
* **A pattern can't contain whitespace**, as it separates the value's parts.

#### Modifiers

A modifier follows the URL it applies to, as `<outcome>:<statuses>`, and says what those statuses mean from that URL, for a source that doesn't answer with a `404` or a file:

* **`not-found`** reads them as no file there, moving on to the next URL: an S3 bucket without list permission answers a missing key with `403`, so its URL takes `not-found:403`.
* **`retry`** reads them as a failure to retry later: jsDelivr sometimes answers `403` for a repository over its size limit, so its `gh/` URLs take `retry:403`.

The statuses are a comma-separated list of `4xx` other than `404`, `408` and `429` (which keep their meaning), with no spaces: a space ends the modifier, so `retry:403, 405` is refused.
A URL takes each outcome at most once, its statuses in one list: `retry:403,405 not-found:402`.

**A source must answer a missing file with `404`**, or with a status a modifier reads as `not-found`: any other `4xx` refuses the icon, and the URLs after it are never tried.

#### How perdu fetches

Perdu fetches each icon once, and keeps it in its [resource cache](how-it-works.md#the-resource-cache) until a refresh or a restart: a new icon appears on the build after the one that first names it, and an icon changed or removed upstream shows as it was fetched until a refresh ([Deploying perdu](deploy.md#refreshing)).

* **A `404`**, another status a modifier maps to `not-found`, or a file over 256 KiB, moves on to the next URL or format; after the last, the icon is *not found*, or refused as too large.
* **Any other `4xx` but `408` and `429`**, or a file served with the wrong media type, refuses the icon until a refresh, without trying the next URL: such an answer is more likely a proxy's or a portal's than the source's.
* **A timeout, a `5xx` or a `408`** is retried at the same URL, backing off to an attempt every 10 minutes; a `429` or a `503` holds back every request to that host, for as long as its `Retry-After` asks, up to 10 minutes, or for the backoff's delay without one.
* **A redirect** is followed only on the URL's own host, to `http` or `https` on any port, allowing upgrades from `http` to `https` but not downgrades; any other case refuses the icon, logged with where it led.
  A redirect chain of more than 10 requests is a failure and retried.

The requests go through `HTTPS_PROXY` and `HTTP_PROXY` when set ([Deploying perdu](deploy.md#fetching-icons-through-a-proxy)).

#### Credentials and `http`

A URL can carry a user and a password, as in `https://user:password@icons.example.com/{name}.svg`, which perdu sends as Basic authentication.
Perdu redacts the password wherever it logs the URL, but not a credential elsewhere in it, such as a token in its query, which is logged as written.

> [!WARNING]
> **Percent-encode a password's `/`, `?` and `#`**, as `%2F`, `%3F` and `%23`: written as they are, each ends the URL's host part, so perdu reads the rest of the password as a port, a path or a query, refuses the URL or sends its requests elsewhere, and logs that part unredacted.

**An `http` URL** sends its requests, and any credential it carries, unprotected, and perdu serves what it gets from its own origin like any other icon.

### Collections

Example remote configurations for well-known collections.

#### Font Awesome Free

```
--icons-remote 'fa:(?<style>solid|regular|brands)-(?<name>.+) https://cdn.jsdelivr.net/npm/@fortawesome/fontawesome-free@7/svgs/{style}/{name}.svg'
```

Font Awesome draws some icons in several styles under one name, so the style is part of the name: `<style>-<name>`, e.g. **`fa:solid-key`, `fa:regular-user`, `fa:brands-github`**.
[Font Awesome's site](https://fontawesome.com/search) lists the free icons and the styles each comes in.
The icons are licensed CC BY 4.0, and each file carries its attribution in a comment, which perdu serves as fetched.

> [!TIP]
> The free styles are `solid`, `regular` and `brands`, to which the pattern restricts the remote.
>
> If you have a Font Awesome subscription, you can mount the Pro icons beside the remote ([Your own icons](#your-own-icons)).
> The download keeps a directory per style, so rename each file with its style: `svgs/light/key.svg` goes to `<icons-dir>/fa/light-key.svg`, and labels name it `fa:light-key`, which your directory resolves before the remote's pattern can refuse.

#### Lucide

```
--icons-remote 'lu:https://cdn.jsdelivr.net/npm/lucide-static@1/icons/{name}.svg'
```

[Lucide](https://lucide.dev/icons/) draws general-purpose icons in a single stroke style, named in lower case with hyphens, e.g. `lu:house`, `lu:circle-check`.
The icons are licensed under the ISC licence, apart from the few Lucide took from Feather, which are under the MIT licence.

#### Tabler Icons

```
--icons-remote 'ti:https://cdn.jsdelivr.net/npm/@tabler/icons@3/icons/outline/{name}.svg'
```

[Tabler Icons](https://tabler.io/icons) are general-purpose icons drawn as outlines, named in lower case with hyphens, e.g. `ti:home`, `ti:brand-github`.
The icons are licensed under the MIT licence.

> [!TIP]
> Many icons also come filled, under the same names, in `icons/filled/`.
> Give them a library of their own, such as `tif:`, with `filled` in place of `outline` in its URL, or put the style in the name, as [Font Awesome's remote](#font-awesome-free) does: `(?<style>outline|filled)-(?<name>.+)`, with `{style}` in place of `outline`.

#### Simple Icons

```
--icons-remote 'si:https://cdn.jsdelivr.net/npm/simple-icons@v16/icons/{name}.svg'
```

[Simple Icons](https://simpleicons.org) holds monochrome brand logos, named in lower case without separators, e.g. `si:github`, `si:homeassistant`.
The remote follows version 16, as Simple Icons' own instructions do: icons are generally removed only in major versions, twice a year, so following the latest, `simple-icons@latest`, can lose an icon, while a remote following one major version gets no new icons once the next is out.
Perdu's page draws these logos in its text colour, as any monochrome icon, so a brand icon doesn't show in its brand's colour ([Icons drawn in the text colour](#icons-drawn-in-the-text-colour)).
Simple Icons publishes its icons under CC0, but the logos are their brands' trademarks, some under licences of their own, and its [disclaimer](https://github.com/simple-icons/simple-icons/blob/develop/DISCLAIMER.md) asks that you read it before using one.

#### Dashboard Icons

```
--icons-remote 'di:https://cdn.jsdelivr.net/gh/homarr-labs/dashboard-icons/{format:svg,png,webp}/{name}.{format} retry:403'
```

The [Dashboard Icons](https://github.com/homarr-labs/dashboard-icons) project holds logos for most self-hosted software, named in lower case with hyphens, e.g. `di:jellyfin`, `di:home-assistant`.
The remote tries SVG, then PNG, then WebP, and follows the repository's default branch, so an icon added upstream is usable soon after it lands; jsDelivr caches a branch's files for up to 12 hours, though, so a new icon can take that long to appear, even after a refresh.
The repository is over jsDelivr's size limit, which makes it answer `403` now and then, hence `retry:403`.
The project is licensed under the Apache License 2.0, and the logos are their owners' trademarks.

The search on dashboardicons.com also returns icons from other collections, marked as *external*, such as selfh.st's, which the `di:` configuration above doesn't resolve.

> [!TIP]
> To use external collections, give them a remote of their own, give the icon's full URL as an `https` icon, or mount a copy ([A copy of a collection](#a-copy-of-a-collection)).

#### selfh.st

```
--icons-remote 'sh:https://cdn.jsdelivr.net/gh/selfhst/icons@main/{format:svg,png,webp}/{name}.{format} retry:403'
```

The [selfh.st icons](https://selfh.st/icons) are logos for self-hosted software, e.g. `sh:navidrome`, licensed CC BY 4.0, which asks that you credit them.
Their repository is over jsDelivr's size limit too.

> [!TIP]
> A second URL makes one collection the fallback of another: a `di:` remote giving dashboard-icons' URL, then selfh.st's, each with `retry:403`, takes dashboard-icons' icon where there is one, and selfh.st's otherwise.
> The two can draw one name differently, as they do with `navidrome`.

## Your own icons

Set `--icons-dir` to a directory you mount into perdu's container, holding **one subdirectory per library** (its name is the library name), and in it **one file per icon** (its name is the icon's, and it must have one of the accepted extensions), for example:

```
/icons/                     --icons-dir=/icons
  logos/
    backup.png              logos:backup
    nas.svg                 logos:nas
  fa/
    light-key.svg           fa:light-key, beside the fa: remote's icons
  di/
    plex.svg                di:plex, replacing the di: remote's
```

A library you mount is usable as soon as its directory is there, with nothing else to configure.
Mounting the directory read-only is fine: perdu only reads it.

**The directory comes first for every library.**
With `--icons-dir` set, perdu resolves each library icon in this order:

1. **the directory**: `<icons-dir>/<library>/<name>`, trying the accepted extensions by priority ([The forms an icon takes](#the-forms-an-icon-takes)).
   A regular file of at most 256 KiB found there is the icon, full stop.
   Anything else found under that name (a larger file, a directory, a link to nothing) is refused with a `WARN`, and **doesn't fall back to the remote**, which would hide a broken override;
2. **only when the directory has no such file**, the library's [remote](#remotes), and the icon is *not found* for a library without one.

A file you mount takes precedence over its library's remote, with no warning, and so does one the remote's pattern refuses: the name is checked against the pattern only after the file is looked up in the directory without finding it.
Perdu moves on from the directory only when it could read the directory and found no file: while the directory itself is missing or can't be read, icons aren't taken from the remotes in its place, and perdu retries.

**File names are matched as your filesystem matches them**, so a reference whose case differs from its file's can resolve on a case-insensitive host (Docker Desktop's bind mounts, typically) and not on Linux: write references in the files' own case.

**Nothing in the directory may lead outside it.**
The directory itself may be a link, as a mount point often is, and a link inside it that stays inside is followed; one leading outside it is refused, so that no file elsewhere on the host can be served under `/assets/icons/`.
Perdu never lists the directory's contents, and reads a library's subdirectory only when a reference names it, so a subdirectory whose name a library name can't have (`~data`, `http`) is never read.
It never looks inside a file either: the extension is the format.

> [!NOTE]
> **Use a local directory**, or a bind mount of one, and **not a network share** such as NFS or SMB.
> A read from a share whose server hangs can wait indefinitely, and perdu can't abandon it: it logs a `WARN` after 5 seconds, and the icons of that read's library wait with it, while the other libraries' go on — until 8 libraries' reads hang, or the directory itself does, after which **every library's icons wait** ([How perdu works](how-it-works.md#the-resource-cache)).
> A local directory can't hang that way.

**Changing a file takes effect on a refresh**, `SIGHUP` ([Deploying perdu](deploy.md#refreshing)), since perdu keeps every icon it has read until then ([How perdu works](how-it-works.md#the-resource-cache)), and doesn't watch the directory: a filesystem's notifications often fail to cross a bind mount.
An icon no page has used yet is read when one first does, so a new service's new icon needs no refresh; replacing a file, adding one perdu found missing, or fixing one it refused does.

At startup the directory has to exist and be readable within the startup window, or perdu doesn't start ([Deploying perdu](deploy.md#starting)).
An empty directory still lets perdu start, but logs a `WARN`, since it usually means a mount that didn't land: a bind mount whose source directory was deleted typically shows up empty, and its icons are then taken from the remotes, where their libraries have one.

### A copy of a collection

To pin a collection, or to never contact its source, mount a copy of it as a library, without a remote, or with one for the icons your copy lacks:

* **dashboard-icons**: mount a copy of its `svg/` directory itself (not the repository's root, which holds a directory per format) at `<icons-dir>/di/`.
* **Font Awesome** doesn't mount as it is: its remote maps `fa:solid-key` to `solid/key.svg`, while your directory needs a flat `solid-key.svg`, so rename each file with its style, as for the Pro icons ([Font Awesome Free](#font-awesome-free)).

A remote can reach one file under several names, by a pattern giving it two, as `solid-key` and `key-solid` above, or by a URL leaving out one of the pattern's groups, while your directory reads each name as a file of its own.

## When an icon is missing

An icon perdu can't resolve leaves its tile showing the default, and each build logs it at `WARN`, `icon unresolved`, naming the reference, the entities using it, the pages, the source that answered and, for a refusal, its cause: a mistyped name reads differently from a proxy's `403`.
For a library with no remote, the line says so, and that `--icons-dir` is unset where it is, so that a forgotten remote doesn't read like a mistyped name.
A source that doesn't answer for a while — your icons directory, or a remote's host — logs `icon source failing` once, and `icon source answering again` when it recovers ([Reading the logs](logs.md#icons)).

## How the page draws icons

**A service without an icon**, or whose icon doesn't resolve, shows a plate holding its initials, up to two letters taken from its name as the page shows it — "Media Server" gives "MS", `home-assistant` "HA".
Each service has one of five colours and one of six shapes, kept for as long as the labels identifying it stay the same ([Labels](labels.md#service-identity)), so two services sharing a name are usually told apart.
**An action without an icon** shows the default action icon ([The template](template.md#icons)).
**An icon the viewer's browser can't load**, e.g. because its host doesn't answer, shows the plate or the default action icon in its place.

### Icons drawn in the text colour

**An icon drawn in one colour takes the page's text colour**: every icon whose visible pixels are all one colour, transparency aside, whether an SVG or a raster image — a Font Awesome icon, a one-colour logo such as `di:github` — so that it reads in both the light and the dark colour schemes, and recedes with an unlinked tile.
That includes **a brand colour**: a logo drawn in a single brand red is drawn in the text colour too.
An icon drawn in several colours keeps them.

A script in the viewer's browser decides, measuring each icon as it shows it on every page view, so an icon changed upstream is measured as it now is; without JavaScript, every icon shows in its own colours.

> [!TIP]
> To check how your icons read in the dark scheme, which a viewer can't switch to from the page, see [Colour schemes](template.md#colour-schemes).

#### Icons on another origin

To measure an icon, the script has to be allowed to read its pixels.
It always can for an icon perdu serves (from a library, a directory you mount, or a `data:` label) and for one at a root-relative path on perdu's host.
For an `http` or `https` icon on another origin, the page loads a second copy of it in CORS mode, without the viewer's cookies or credentials, and measures that copy when the icon's host allows it with `Access-Control-Allow-Origin`.

* **A host that refuses CORS** leaves its icon unmeasured, shown in its own colours.
* **A host behind an authenticating proxy**, such as the one in front of perdu, refuses the request as it carries no credentials, on every page view: a refused request per icon per view, and the icon shown as it is.

Either is avoided by **serving the icon from perdu** — through a remote, from a directory you mount, or as a `data:` label — or by letting the proxy pass the icon's path through without login and with `Access-Control-Allow-Origin: *`.

### Safety

The page loads every service's and action's icon as an image, never as markup, so an SVG's scripts never run in it.

Responses under `/assets/icons/` also carry `Content-Disposition: attachment`, so that a viewer opening an icon's address directly downloads it, and `Content-Security-Policy: sandbox` and `X-Content-Type-Options: nosniff`, so that a browser showing it anyway runs nothing it holds; a policy your proxy sets there has to keep perdu's ([Deploying perdu](deploy.md#content-security-policy)).

An `http` or `https` icon makes every viewer's browser contact the server it names, and a root-relative one sends a request to perdu's host carrying the viewer's cookies; perdu's own paths do nothing for such a request beyond answering it.
