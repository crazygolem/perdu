# Changelog

Perdu's release notes, newest first: for each version, what changed for whoever runs perdu, and what to do when upgrading.

## 0.2.0 — 2026-11-15

* **Added configurable icon library remotes, and removed built-in icon libraries**: The `fa` and `di` icons which were previously built into perdu will stop resolving until you configure the equivalent library remotes (or add them to the mounted icons directory).
  [Collections](https://crazygolem.github.io/perdu/icons/#collections) gives the remotes for both.
* **Added support for several services per container**, lifting 0.1.0's limit of one: a container's labels can declare further services under keyed blocks, each a tile of its own ([Several services on one container](https://crazygolem.github.io/perdu/labels/#several-services-on-one-container)).
  Upgrading may change each service's initials plate colour and shape.
* **Renamed `--identity-labels` to `--workload-labels`**, and `PERDU_IDENTITY_LABELS` to `PERDU_WORKLOAD_LABELS`: rename them when upgrading, since perdu refuses the old flag as unknown, and ignores the old variable with a `WARN`, falling back to the default keys.
* **Changed the log lines about labels and entities**: a line names a workload, a service or an action by its identity, never by its name ([What a line names](https://crazygolem.github.io/perdu/logs/#what-a-line-names)).
  `label ignored` is now `container labels ignored` or `labels ignored`, `service left off` is `services left off`, and a workload's services set aside are logged as `services excluded`: a log collector matching the old lines must follow.
* **Actions without a usable `url` aren't shown anymore**, whether nested or on a tile of their own, where 0.1.x drew them dimmed; perdu still logs a `WARN` for a `url` it can't use ([Actions](https://crazygolem.github.io/perdu/template/#actions)).
* **Reworked how receding tiles are indicated under more contrast and forced colours**: the *link-off* icon, a chain link with a slash through it, marks tiles that link nowhere and otherwise show dimmed ([Other system preferences](https://crazygolem.github.io/perdu/template/#other-system-preferences)).
  Under more contrast, 0.1.x marked them with a hatched mask over their icon, which made some icons barely legible.
* Built with Go 1.27.4, which fixes [GO-2026-6859](https://pkg.go.dev/vuln/GO-2026-6859), a crash of Go's HTTP/1 server.

## 0.1.1 — 2026-11-06

* Added missing licences to the image's `/licenses/` directory and the public repository's `NOTICES` file.

## 0.1.0 — 2026-11-05

MVP release:

* **Services from container labels:** perdu reads labels from your docker containers and lists the services they describe, rebuilding its pages whenever a container changes.
* **Tiles, sections and actions:** each service gets a tile with its name, icon, description and link; categories group tiles into sections, with an administration section of its own; a service's actions are listed in its tile, or get a tile of their own in another section.
* **Multilingual:** a page per language; labels can be translated, right-to-left languages are supported, and viewers land on the page their browser or their earlier choice asks for, with a switcher to change it; the interface comes in English, French, German and Italian.
* **Per-viewer touches:** behind an authenticating reverse proxy, expressions over the request's headers let the page greet each viewer and hide the tiles not meant for them. Hiding is presentational, not access control: every page still holds every tile.
* **Icons:** Font Awesome Free built in, dashboard-icons fetched from a CDN and cached, your own from a directory, URLs and `data:` icons; a service without one gets its initials on a coloured plate, an action a bolt.
* **For every viewer:** light and dark colour schemes, following the system's contrast, motion and text size settings; targets WCAG 2.2 level AA; stays usable without JavaScript.
* **Simple to run:** one binary in a non-root image for `linux/amd64` and `linux/arm64`, needing only read access to the docker API (tested with Docker and Podman), with logs in text or JSON.

Limitations of the MVP:

* One service per container.
* Only containers from one docker daemon are listed; there is no support for alternative providers.
* No running state: a stopped service's tile looks like any other.
* Rudimentary service identity via a single list of labels (`--identity-labels`).
* Only two icon libraries have built-in support, Font Awesome and dashboard-icons; any other has to be mounted as a library of your own.
* Behind a reverse proxy, the client's address and scheme are read only from `X-Forwarded-For` and `X-Forwarded-Proto`: `Forwarded` and `X-Real-IP` aren't supported.
* Basic UI:
  * No search.
  * No light/dark switch on the page.
  * Categories only group tiles: their names aren't shown, so sections have no headings, the administration section aside.
* No page customization:
  * The order can't be chosen: sections follow their category values, tiles the alphabet.
  * The template and its look are fixed, as are the page's title and heading: no theming.
  * The interface can't be translated beyond English, French, German and Italian, and the default language must be one of them.
* No compatibility promise before 1.0: flags, labels, expressions, cookie names and log lines may change in any release.

The manual: <https://crazygolem.github.io/perdu/>
