# Changelog

Perdu's release notes, newest first: for each version, what changed for whoever runs perdu, and what to do when upgrading.

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
