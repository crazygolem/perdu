# Labels

Perdu builds its page from labels on your containers: whoever writes a service's labels decides what its tile offers, and whoever configures perdu decides what the page shows and how.

Every label perdu reads starts with the namespace, `perdu` by default and configurable via [`--namespace`](flags.md#--namespace), written `<prefix>` below.

## Declaring a service

A container is on the page when it carries `<prefix>.enable=true` and a service name:

```yaml
services:
  grafana:
    image: grafana/grafana
    labels:
      perdu.enable: "true"
      perdu.name: Grafana
      perdu.url: https://grafana.example.com
```

**Perdu lists only the containers carrying `<prefix>.enable`**, so that label has to be spelled exactly: a container whose `enable` label is mistyped is left off the page without a warning, while every other mistyped label under the prefix is warned about.
Perdu also lists stopped containers, so a stopped service stays on the page ([Service identity](#service-identity)).

## The labels

| Label                             | Required        | Localizable | Holds                                                                |
| --------------------------------- | --------------- | ----------- | -------------------------------------------------------------------- |
| `<prefix>.enable`                 | yes             | no          | whether perdu reads the container's labels: `true` or `false`        |
| `<prefix>.name`                   | yes             |             | the service's name                                                   |
| `<prefix>.description`            |                 |             | the tile's second line                                               |
| `<prefix>.url`                    |                 |             | where the tile links                                                 |
| `<prefix>.icon`                   |                 |             | the tile's icon ([Icons](icons.md))                                  |
| `<prefix>.category`               |                 | no          | the sections the tile is in, comma-separated                         |
| `<prefix>.audience`               |                 | no          | the groups the tile is shown to, comma-separated; empty is everybody |
| `<prefix>.actions.<key>.name`     | for each action |             | an action's name                                                     |
| `<prefix>.actions.<key>.icon`     |                 |             | an action's icon                                                     |
| `<prefix>.actions.<key>.url`      |                 |             | where an action links                                                |
| `<prefix>.actions.<key>.category` |                 | no          | an action's sections; if empty, takes the service's                  |
| `<prefix>.actions.<key>.audience` |                 | no          | the groups an action is shown to; never falls back to the service's  |

A service and each of its actions are **entities**: each is declared by its labels, shown on its own, and set aside on its own when a label of its own can't be used ([What happens to a label perdu can't use](#what-happens-to-a-label-perdu-cant-use)).
An action has no description.

A service's `name`, `description`, `url` and `icon`, and an action's `name`, `url` and `icon`, can be given per language by adding a locale to the key, as in `<prefix>.name.fr=Fou` ([Locales](locales.md#translating-labels)); `enable`, `category` and `audience` can't.

## How a value is read

**Every value is trimmed** of the spaces around it before anything reads it, so a value of only spaces is an empty one.

> [!NOTE]
> This is in contrast with flags, whose value is never trimmed as a whole ([Flags](flags.md#giving-flags)); perdu is generally more lenient for entity definitions than it is for its own configuration.

**An empty value is a value, and only a label that isn't present is none.**
`<prefix>.description=` gives the service no description, where leaving the label out lets a translated page fall back to another language's description ([Locales](locales.md#falling-back)); an empty value asks for no page, so `<prefix>.description.sv=` alone generates no Swedish page ([Locales](locales.md#which-pages-exist)).
An optional label left empty is never warned about: `<prefix>.icon=` gives the tile no icon, quietly.

**A list** — `category` and `audience` — is comma-separated, each element trimmed, so `user, admin` is `user` and `admin`, while `Media Server` is one category.
An element given twice counts once, and the order you wrote them in is kept (first occurrences win).
There is no way to escape a comma, so an element can't hold one.
**An element left empty**, as in `audience=,` or `category=Media,,Tools`, is a mistake rather than an element to drop, and sets its entity aside with a `WARN`: `audience=,` would otherwise publish a service to everyone.
An empty list, `<prefix>.audience=` or `<prefix>.category=`, declares nothing, as leaving the label out does.

**A key is matched exactly as written**, its locale apart: `<prefix>.Name` is not `<prefix>.name`, and `<prefix>.actions.Foo.name` and `<prefix>.actions.foo.name` are two actions.
A misspelled key is ignored with a `WARN` naming it, and its service may then be set aside for the label it misses, with a second `WARN`.

### `enable`

`<prefix>.enable` says whether perdu reads the container's other labels at all, and declares nothing by itself: a container carrying only `enable` puts nothing on the page.
It takes `true` or `false`, and the other spellings Go reads as booleans: `1`, `t`, `T`, `TRUE`, `True`, and `0`, `f`, `F`, `FALSE`, `False`.

* `false` switches the container off without removing its labels, silently.
* **An empty value is refused**, as is anything outside that set, and sets the service aside with a `WARN`: writing the bare key, as in `labels: ["perdu.enable"]`, could mean on or off, so perdu asks you to write which.
* `enable` never takes a locale; `<prefix>.enable.fr` is ignored with a `WARN`.

### `name`

An entity's name, shown as its tile's or its line's heading.
It must be non-empty wherever it is given, and must have a value the page falls back to — `<prefix>.name`, or `<prefix>.name.<default-locale>` — so that every page has one ([Locales](locales.md#falling-back)); otherwise the entity is set aside with a `WARN`.

### `description`

The tile's second line, at most five lines on the page ([The template](template.md#long-text)).
Without one, the tile shows its URL there.

### `url`

Where a tile or an action links.
Without a usable one, it links nowhere, and the page shows it dimmed ([The template](template.md#links)).

Perdu accepts two forms:

* **an absolute URL with an `http`, `https` or `mailto` scheme**, an `http` or `https` one naming a host: `https://jellyfin.example.com`, `mailto:admin@example.com`.
  No other scheme is accepted (`ssh:`, `vnc:`, `obsidian:`, etc.) because perdu keeps Go's template escaping rather than bypassing it.
  That escaping turns any other scheme into an inert link, which also keeps a `javascript:` link harmless, whoever wrote the label;
* **a root-relative path**, with no scheme or host, beginning with `/` and holding no backslash: `/plex/`, for a service served on a path of the same host as perdu.

Anything else is **unusable**: another scheme, a path not beginning with `/` such as `plex`, which would resolve against the page's own address `/l/en/`, or a value perdu can't parse.
An unusable value isn't an error: its tile stays, linking nowhere, and a `WARN` naming the label says why on every build until you fix it.

**Perdu parses a URL more strictly than a browser**, so a URL your browser follows can still be unusable here.
These are the spellings it refuses, and the forms that work:

| Refused                                                                                   | Write instead                                |
| ----------------------------------------------------------------------------------------- | -------------------------------------------- |
| a `%` that starts no escape, in the path or after `#`: `https://example.com/100%`         | `%25`: `https://example.com/100%25`          |
| a tab or another control character                                                        | its escape, `%09` for a tab                  |
| a space in the host: `https://exa mple.com/`                                              | the host without it                          |
| an escape spelling a letter, a digit or other ASCII in the host: `https://ex%41mple.com/` | the character itself: `https://exAmple.com/` |
| a port that isn't a number: `https://example.com:port/`                                   | the port's number                            |
| an IPv6 host missing its `]`: `https://[::1/`                                             | `https://[::1]/`                             |
| `https:example.com`, which names no host                                                  | `https://example.com`                        |
| `//host/path`, which names a host in a root-relative form                                 | `https://host/path`                          |
| a backslash in a root-relative path: `/a\b`                                               | `/a/b`                                       |

A `%` in the query, after `?`, is taken as written.

### `category`

The sections a tile is in: each value names a section, and a tile is drawn once in every section its entity names.
Perdu takes a value as a key that groups tiles, never as text to show: **the built-in template shows no category value anywhere**, telling its sections apart by a line between them.

```
perdu.category=Media
perdu.category=Media,Downloads
```

Unlike `audience`, a category value takes any character but a comma.
In this release the named sections are grouped by exact matching (`Media` and `media` are two sections) and follow one another in the code-point order of their category values, the same on every page; that order is kept only while sections show no heading, so don't build on it.
A category can't take a locale: `<prefix>.category.fr` sets its entity aside with a `WARN`.

**A value starting with `~` names a placement** rather than a section: a place the page's template defines.
The built-in template defines two:

* **`~default`**, the uncategorized section, which comes **first**, before the named sections.
  A stack that sets no `category` at all shows one section holding everything; as you name categories, the still-uncategorized tiles stay above them.
  A **service** whose category is left **empty or unset** is placed at `~default`, as if it had declared it.
  For an **action**, an empty or unset category makes it take **its parent service's**.
* **`~admin`**, the administration section, which comes **last**.
  It is a place for what most viewers use least, and **it places without restricting**: `audience` defines who sees a tile there, as anywhere else.
  To show an administration service to administrators only, pair the two:

  ```
  perdu.category=~admin
  perdu.audience=admins
  ```

  An action placed there under a service placed elsewhere is a tile of its own in it.

Placements compose with sections: `category=~default,Media` puts a tile in both.
Any other placement, a mistyped one such as `~admni` included, shows nowhere in the built-in template, and every build logs an `INFO` line naming the placement and the entities placed there.

### `audience`

The groups a tile is shown to: a viewer sees it when **any one** of their groups is among its values.

```
perdu.audience=admins
perdu.audience=family,friends
```

> [!WARNING]
> **Hiding a tile is presentation, not access control.**
> Every page holds every tile, whoever views it: a viewer without JavaScript, or reading the page's source, sees them all.
> A service has to protect itself.

**Each value is a token**, and must match `[a-zA-Z0-9._-]+`; identity-provider groups such as `team.admins` work.
A value holding anything else (including spaces) sets its entity aside with a `WARN`, rather than being dropped: dropping the only value would show the tile to everyone.
An audience can't take a locale: `<prefix>.audience.fr` sets its entity aside with a `WARN`.

**A tile with no `audience` is shown to everyone.**
There is no way to show a tile to nobody: an empty list declares no restriction, as leaving the label out does.

> [!TIP]
> `audience` declares restrictions, no `audience` means no restriction.

A viewer's groups come from `--audience-expr` ([Per-viewer data](rules.md#the-viewers-groups)).
With no rule computing them, every viewer sees every tile, whatever its `audience`.
Each build then logs an `INFO` line naming the entities that declare one, since their labels ask for filtering that no rule provides.

> [!TIP]
> A rule that gives a viewer no groups is not the same as no rule: that viewer sees only the tiles without an `audience`.
>
> To show a tile to viewers with no groups, have your `--audience-expr` give them a group of their own, such as `guest`, and list it in `audience`.

### Actions

An action is a link a service offers beside its own, such as a settings page or a "Request access" form, declared under `<prefix>.actions.<key>`:

```
perdu.actions.password.name=Change password
perdu.actions.password.url=https://auth.example.com/password
perdu.actions.password.icon=fa:solid-key
```

**The key** identifies the action among its service's, and is never shown.
It must match `[a-zA-Z0-9_-]+`, and an action whose key holds anything else is set aside with a `WARN`, leaving its service in place.
A service's actions show in the code-point order of their keys: digits before capitals before lower case.

> [!TIP]
> Use keys such as `1-sessions` and `2-password` to set an order independent of the words in them.

**An action is nested or standalone, section by section.**
In a section that also holds its service, it is drawn inside the service's tile, as a line under its head; in a section that doesn't, it is a tile of its own, which shows its service's name as its second line.
An action that declares no category takes its service's, so it is nested in each of its service's sections; one declaring a category its service doesn't share has a tile of its own in that section.

> [!TIP]
> To put an action in the uncategorized section while its service has categories, use `category=~default`.

**An action's audience is its own.**
Nested, it is hidden whenever its service is, whatever it declares, since it is drawn inside the service's tile; standalone, nothing contains it, so it is restricted by its own `audience` alone.

> [!TIP]
> An administrators-only service can offer a "Request access" action that everyone sees, in a section the service isn't in:
>
> ```
> perdu.category=Admin tools
> perdu.audience=admins
> perdu.actions.access.name=Request access
> perdu.actions.access.url=https://example.com/access
> perdu.actions.access.category=Tools
> ```

## Service identity

Several containers can declare the same service: most often the replicas of a scaled compose service.

Perdu groups containers carrying `<prefix>.enable` by the values of the labels `--identity-labels` lists, by default Compose's project and service labels, so a compose service is one service however many containers it runs ([How perdu works](how-it-works.md#what-a-build-does)).

* **A container missing any of those identity labels is never on the page**, and unless its own `enable` is `false`, perdu logs an `INFO` line naming it and the labels it lacks.
  A container you start with `docker run -l perdu.enable=true`, outside Compose, lacks the Compose labels the default lists.
* **A service's containers rank by state**: running first, then unavailable, then stopped, and only those of the highest rank present define it.
  So a stale stopped container can't hide the one running, and a replacement restarting in a loop leaves the running container's labels on the page; when none is live (running or unavailable), the stopped ones define the service and it stays on the page.

  In order to limit the staleness of the information displayed on a page and avoid flicker, the ranks have different rules for picking the defining containers:

  | Rank | State                                               | Defines a service    |
  | ---- | --------------------------------------------------- | -------------------- |
  | 1    | `running`                                           | The oldest container |
  | 2    | unavailable: `paused`, `restarting`                 | The oldest container |
  | 3    | stopped: `created`, `exited`, `dead`, being removed | The newest container |

  If there is a tie, i.e. several oldest or newest containers created in the same second, conflicting labels set the service aside with a `WARN` naming them, unless they all set `enable` to `false` (whatever else they declare): the service is then off, as one container switching it off would leave it.
  Otherwise, perdu names the containers of that rank whose labels differ from those defining the service: at `INFO` while the service is on the page, or where a newer live container would switch on a service that is off, so that a service you just enabled or fixed says why it doesn't show yet; at `DEBUG` otherwise, for a service left off whatever the container declares.

  During a redeploy, the new live container waits for the old one to stop (it counts as running until its process exits); a new container still ignored after that differs from another live container of its service, older than itself.
  The older one may be a one-off container run from the service's definition before the redeploy (such as one `docker compose run` starts), which keeps the old labels on the page while it runs; otherwise, the difference is a mistake worth fixing.
  Once the service has stopped, the container ignored is an older one left behind, which you can remove.

* **A service defined only by stopped containers gets one more check.**
  Perdu then looks for a live container with the same identity that carries no `<prefix>.enable` — what a redeploy that removed the service's labels leaves — and if it finds one, leaves the service off the page (with an `INFO` line) so that removing a service's labels removes its tile.
  Each such service costs an extra docker call per build.

With `--identity-labels` empty, every container is a service of its own, identified by its container ID, and none of the above applies.

Starting or stopping a container starts no build, so that a container restarting in a loop doesn't rebuild the page every few seconds; where it changes which containers define a service, the page follows on the next build, or on a refresh ([Deploying perdu](deploy.md#refreshing)).
On the other hand, a container carrying `<prefix>.enable` that perdu ignores, or whose service it sets aside, still starts a build when it is created, updated or removed, and that build runs to completion even though it produces the same pages ([How perdu works](how-it-works.md#when-a-build-runs)).

## Labels an image declares

**A container's labels include those its image declares**, and perdu reads them as the container's own: image and container labels are merged by docker and indistinguishable when read from containers.

An image declaring `<prefix>.*` labels therefore puts itself on the page as they describe — and can add locales, make every viewer's browser contact a server it names, make perdu fetch `di:` icons it names from jsDelivr, and declare another service's identity labels beside perdu labels of its own, which can put its labels on that service's tile or set the service aside ([Service identity](#service-identity)).

You can override an image's labels on the container, but not remove them; Compose's `!reset` doesn't reach them.
The levers are `<prefix>.enable=false` on the container, a [`--namespace`](flags.md#--namespace) the image doesn't use, [restricting generated locales](locales.md#which-pages-exist), and not running the image.

## When perdu checks a label

Perdu checks your labels in two steps, on every build.

**Reading the labels** decides what exists: which services and actions stand, which pages there are, and the value each page shows for each field, its own locale's or the one it falls back to ([Locales](locales.md#falling-back)).
A problem found here sets an entity aside or ignores a key.

**Rendering the pages** checks the values chosen for each page: a `url`'s scheme and shape, an icon's form, whether a library has the icon.
A value that fails leaves its field empty on the pages that show it, and changes nothing else: the entity and the page stay ([Locales](locales.md#falling-back)).

## What happens to a label perdu can't use

Perdu never fails the page over a label.
Each problem costs as little as it can, and is logged at `WARN` on every build while it lasts, once per build per label, naming the label, the entity carrying it and the pages it affects, so that it stays in recent logs.

**These set an entity aside**, logged as `service excluded` or `action excluded`; a service set aside takes its actions with it, while an action set aside leaves its service in place; **neither remove locales**:

* a `name` left empty, or with no value to fall back to;
* an `enable` that is empty or isn't a boolean, a mix of `true` and `false` among a service's containers, or a service's containers declaring different labels;
* an action key outside its characters, which sets that action aside;
* an `audience` value outside its characters;
* an empty element in a `category` or `audience` list;
* a `category` or `audience` key with a locale;
* two keys for one field whose locales are one locale spelled two ways, as `<prefix>.name.fr-ch` and `<prefix>.name.fr-CH`, which sets aside the entity carrying the field.

**These are ignored, and the entity stays**, logged as `label ignored`: a key perdu doesn't know under the prefix; a locale it can't use ([Locales](locales.md#translating-labels)); a locale on `enable`.

**These leave a value empty, and the entity stays**, logged as `unusable value`: a `url` or an `icon` perdu can't use, and a `data:` icon it can't decode ([Icons](icons.md)); and logged as `icon unresolved`, an icon reference no library answers.

Nothing is logged for a label left out, an optional label left empty, or `enable=false`.
