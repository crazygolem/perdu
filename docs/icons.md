# Icons

Services and actions show an icon, configured with `<prefix>.icon` and `<prefix>.actions.<key>.icon`.
Without one, the page draws a default: for services their initials on a coloured plate, and for actions a bolt ([How the page draws icons](#how-the-page-draws-icons)).

## The forms an icon takes

| Form                               | Example                                       | What perdu does                                                                              |
| ---------------------------------- | --------------------------------------------- | -------------------------------------------------------------------------------------------- |
| a library icon, `<library>:<name>` | `fa:solid-key`, `di:jellyfin`, `logos:backup` | resolves it from that library ([Libraries](#libraries)) and serves it under `/assets/icons/` |
| an `http` or `https` URL           | `https://jellyfin.example.com/favicon.ico`    | nothing: the viewer's browser fetches it                                                     |
| a root-relative path               | `/plex/favicon.ico`                           | nothing: the viewer's browser fetches it from perdu's host                                   |
| a `data:` URI                      | `data:image/png;base64,iVBORw0…`              | decodes it, and serves it under `/assets/icons/` ([`data:` icons](#data-icons))              |

* **A library icon**'s library and name must each match `[a-zA-Z0-9_-]+`, and are case sensitive: `fa` is the built-in library, and `FA` is not.
* **An `http` or `https` URL** must name a host.
  Perdu never fetches it, and never knows whether it loads; it suits your stack's services' own favicons.
* **A root-relative path** begins with `/`, for a service served on a path under perdu's own host.
* A value starting with `http:`, `https:` or `data:`, in any case, is always that URI form, so no library can be named `http`, `https` or `data`.
* `http`, `https` and root-relative icons are parsed as strictly as a [`url`](labels.md#url).

**Perdu checks the icons it serves**, whatever the source; `http`, `https` and root-relative icons never pass through perdu, and go unchecked.
Checked icons must be **at most 256 KiB**, and declare one of the accepted formats by their extension or media type, tried by priority wherever perdu has a choice:

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

### `fa`: Font Awesome

Font Awesome Free's icons are built into perdu, so they need no network, and without `--icons-dir` they show on the very first page.

Font Awesome draws some icons in several styles under one name, so in perdu the style is part of the name: `<style>-<name>`, e.g. **`fa:solid-key`, `fa:regular-user`, `fa:brands-github`**.
[Font Awesome's site](https://fontawesome.com/search) lists the free icons and the styles each comes in.
The icons are licensed CC BY 4.0.

> [!TIP]
> The styles available with Font Awesome's Free icons are `solid`, `regular` and `brands`.
> The Pro styles are not built into perdu.
>
> If you have a Font Awesome subscription, you can download and mount the Pro icons to make them available to perdu ([Your own icons](#your-own-icons)).
> The download keeps a directory per style, so rename each file with its style: `svgs/light/key.svg` goes to `<icons-dir>/fa/light-key.svg`, and labels name it `fa:light-key`.

### `di`: dashboard-icons

The [dashboard-icons](https://github.com/homarr-labs/dashboard-icons) project holds logos for most self-hosted software, named in lower case with hyphens, e.g. `di:jellyfin`, `di:home-assistant`.

Perdu fetches them from jsDelivr's CDN, at `https://cdn.jsdelivr.net/gh/homarr-labs/dashboard-icons/<format>/<name>.<format>`, trying SVG, then PNG, then WebP, and taking the first that exists.
It fetches the latest version rather than a pinned one, so an icon added upstream is usable the day it lands.
It fetches each icon once, and keeps it until a refresh or a restart, as it keeps every library icon in its [resource cache](how-it-works.md#the-resource-cache): a new `di:` icon appears on the build after the one that first names it, and an icon changed or removed upstream shows as it was until a refresh ([Deploying perdu](deploy.md#refreshing)).
While the CDN can't be reached, perdu keeps retrying, backing off to an attempt every 10 minutes.
The fetches go through `HTTPS_PROXY` when set ([Deploying perdu](deploy.md#fetching-icons-through-a-proxy)).

Three behaviours of third parties are worth knowing, as perdu meets them rather than as guarantees:

* **A `403`**, from a corporate proxy or from the CDN, is taken as a refusal and kept until a refresh, like any refusal: once the proxy lets the request through, send perdu `SIGHUP`.
* **A redirect** is followed only to an `https` address on jsDelivr's own host: any other is taken as a refusal, as an answer that isn't an image is, and logged with where it led.
* **jsDelivr caches** the paths perdu fetches for a while, so an icon published upstream can take time to appear, even after a refresh.

To pin or override `di:` icons, mount a copy of dashboard-icons' `svg/` directory itself (not the repository's root, which holds a directory per format) at `<icons-dir>/di/`.

> [!TIP]
> **On a host that must never contact the CDN**, mount a copy of dashboard-icons' `svg/` directory as a library of its own under **a name other than `di`**, such as `<icons-dir>/dashboard-icons/`, and use `dashboard-icons:<name>` instead of `di:<name>` for all icon labels.
>
> A custom library you mount is read from the mount only, whereas a `di:` name your copy lacks would still be fetched from the CDN ([Your own icons](#your-own-icons)).
>
> Image labels can also set `di:` icons, and perdu can't distinguish those labels from the container's; you have to explicitly override them ([Labels an image declares](labels.md#labels-an-image-declares)).

#### Icons dashboardicons.com finds elsewhere

The search on dashboardicons.com can return icons from other sources, marked as *external* (e.g. the selfh.st collection's), which `di:` doesn't resolve: `di:` with such a name finds nothing.
To use them, either give their full URL (as an `https` icon), or mount a copy as a library of its own.

## Your own icons

Set `--icons-dir` to a directory you mount into perdu's container, holding **one subdirectory per library** (its name is the library name), and in it **one file per icon** (its name is the icon's, and it must have one of the accepted extensions), for example:

```
/icons/                     --icons-dir=/icons
  logos/
    backup.png              logos:backup
    nas.svg                 logos:nas
  fa/
    solid-key.svg           fa:solid-key, replacing Font Awesome's
  di/
    plex.svg                di:plex, replacing dashboard-icons'
```

A library you mount is usable as soon as its directory is there, with nothing else to configure.
Mounting the directory read-only is fine: perdu only reads it.

**The directory comes first for every library.**
With `--icons-dir` set, perdu resolves each library icon in this order:

1. **the directory**: `<icons-dir>/<library>/<name>`, trying the accepted extensions by priority ([The forms an icon takes](#the-forms-an-icon-takes)).
   A regular file of at most 256 KiB found there is the icon, full stop.
   Anything else found under that name (a larger file, a directory, a link to nothing) is refused with a `WARN`, and **doesn't fall back to the built-in icon**, which would hide a broken override;
2. **only when the directory has no such file**, the library's own source: Font Awesome's built-in icons for `fa`, the CDN for `di`, and nothing for any other library.

A library you mount takes precedence over the built-in one of the same name, `fa` included, with no warning.
Perdu moves on from the directory only when it could read the directory and found no file: while the directory itself is missing or can't be read, icons aren't taken from the built-in libraries or the CDN in its place, and perdu retries.

**File names are matched as your filesystem matches them**, so a reference whose case differs from its file's can resolve on a case-insensitive host (Docker Desktop's bind mounts, typically) and not on Linux: write references in the files' own case.

**Nothing in the directory may lead outside it.**
The directory itself may be a link, as a mount point often is, and a link inside it that stays inside is followed; one leading outside it is refused, so that no file elsewhere on the host can be served under `/assets/icons/`.
Perdu never lists the directory's contents, and reads a library's subdirectory only when a reference names it, so a subdirectory whose name a library name can't have (`~data`, `http`) is never read.
It never looks inside a file either: the extension is the format.

> [!NOTE]
> **Use a local directory**, or a bind mount of one, and **not a network share** such as NFS or SMB.
> A read from a share whose server hangs can wait indefinitely, and perdu can't abandon it: it logs a `WARN` after 5 seconds, and the icons queued behind that read wait with it — **those of every library**, once the server hangs ([How perdu works](how-it-works.md#the-resource-cache)).
> A local directory can't hang that way.

**Changing a file takes effect on a refresh**, `SIGHUP` ([Deploying perdu](deploy.md#refreshing)), since perdu keeps every icon it has read until then ([How perdu works](how-it-works.md#the-resource-cache)), and doesn't watch the directory: a filesystem's notifications often fail to cross a bind mount.
An icon no page has used yet is read when one first does, so a new service's new icon needs no refresh; replacing a file, adding one perdu found missing, or fixing one it refused does.

At startup the directory has to exist and be readable within the startup window, or perdu doesn't start ([Deploying perdu](deploy.md#starting)).
An empty directory still lets perdu start, but logs a `WARN`, since it usually means a mount that didn't land: a bind mount whose source directory was deleted typically shows up empty, and its icons are then taken from the built-in libraries.

**After a restart, the first page has no mounted or fetched icons**, and with `--icons-dir` set, no Font Awesome icon either, since every library icon is then read from the directory first ([How perdu works](how-it-works.md#the-resource-cache)).
They arrive on the build that follows a moment later: reload the page to see the icons.

## When an icon is missing

An icon perdu can't resolve leaves its tile showing the default, and each build logs it at `WARN`, `icon unresolved`, naming the reference, the labels using it, the pages, the source that answered and, for a refusal, its cause: a mistyped name reads differently from a proxy's `403`.
A library that doesn't answer for a while — a hung directory, an unreachable CDN — logs `icon source failing` once, and `icon source answering again` when it recovers ([Reading the logs](logs.md)).

## How the page draws icons

**A service without an icon**, or whose icon doesn't resolve, shows a plate holding its initials, up to two letters taken from its name as the page shows it — "Media Server" gives "MS", `home-assistant` "HA".
Each service has one of five colours and one of six shapes, kept for as long as the labels identifying it stay the same ([Labels](labels.md#service-identity)), so two services sharing a name are usually told apart.
**An action without an icon** shows a bolt.
**An icon the viewer's browser can't load**, e.g. because its host doesn't answer, shows the plate or the bolt in its place.

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

Either is avoided by **serving the icon from perdu** — from a directory you mount, or as a `data:` label — or by letting the proxy pass the icon's path through without login and with `Access-Control-Allow-Origin: *`.

### Safety

The page loads every service's and action's icon as an image, never as markup, so an SVG's scripts never run in it.

Responses under `/assets/icons/` also carry `Content-Disposition: attachment`, so that a viewer opening an icon's address directly downloads it, and `Content-Security-Policy: sandbox` and `X-Content-Type-Options: nosniff`, so that a browser showing it anyway runs nothing it holds; a policy your proxy sets there has to keep perdu's ([Deploying perdu](deploy.md#content-security-policy)).

An `http` or `https` icon makes every viewer's browser contact the server it names, and a root-relative one sends a request to perdu's host carrying the viewer's cookies; perdu's own paths do nothing for such a request beyond answering it.
