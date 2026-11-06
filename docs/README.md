# Perdu operator manual

Perdu is a landing page for self-hosted application stacks.

It reads labels from docker containers, builds a static page per language listing the discovered services, and serves those pages.

A small expression engine lets you send dynamic data alongside the static page, so that scripts can customize the rendered page for each user or request, e.g. hiding services from users depending on their groups.

Perdu does not handle authentication; it is expected to be deployed behind an authenticating reverse proxy, which forwards trusted inputs to perdu for user-based customizations.

> [!IMPORTANT]
> Perdu is not appropriate for hiding sensitive services from users: the backend serves a static page always listing all discovered services; the hiding mechanism is only presentational.
>
> Perdu does not perform any kind of access control; the services it links to must be separately protected.

This manual is for whoever runs perdu: what you need to deploy it, configure it and read what it reports.

## Contents

| Chapter                                     | What it covers                                                                                                                                        |
| ------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------- |
| [Deploying perdu](deploy.md)                | The image, a compose stack, access to the docker API, the reverse proxy in front of perdu, starting, refreshing and stopping, and non-standard setups |
| [Flags and environment variables](flags.md) | Every flag, its environment variable, its default and the values it refuses                                                                           |
| [Labels](labels.md)                         | Every label, how containers become services, and what happens to a label perdu can't use                                                              |
| [Locales](locales.md)                       | Which pages exist, translating labels, the language switcher, and how a viewer lands on a page                                                        |
| [Icons](icons.md)                           | The forms an icon can take, the icon libraries, your own icons, and how the page draws them                                                           |
| [Per-viewer data](rules.md)                 | `--data` and `--audience-expr`: the expressions, what they can read, what they produce, and their limits                                              |
| [The template](template.md)                 | What the built-in template shows, and how it behaves for a viewer                                                                                     |
| [Reading the logs](logs.md)                 | The log's levels, its lines, and what each tells you                                                                                                  |
| [How perdu works](how-it-works.md)          | Perdu's parts, and how labels, icons and requests make their way to the page: builds, the resource cache, refreshes and requests                      |

## How perdu works, in brief

Perdu works on three fronts.

**Builds** turn container labels into pages.
A build lists the containers carrying perdu's `enable` label, groups them into services, reads their labels, and renders one page per language into memory, replacing the pages it served before.
A build runs at startup, whenever a container is created, updated or removed, and when you ask for one with `SIGHUP` ([Deploying perdu](deploy.md#refreshing)).
A label perdu can't use costs the service or the value carrying it, never the whole page.

**Requests** are answered from those pages.
Perdu never changes a page per viewer: a page request runs your expressions and sets their results as cookies beside the page.

**Pages** run scripts in the viewer's browser, which read those cookies to customize the rendered page; e.g. to hide services not meant for the viewer.

[How perdu works](how-it-works.md) explains each front further: perdu's parts, and how a label, an icon and a request each make their way to the page.

## Conventions

* `<prefix>` stands for the namespace set with `--namespace`, `perdu` unless you change it: `<prefix>.name` is the label `perdu.name`.
* A path such as `/l/en/` is relative to the base path set with `--base-path`: under `--base-path=/dashboard`, it is `/dashboard/l/en/`.

> [!IMPORTANT]
> **Nothing in this documentation is a compatibility promise yet.**
>
> Perdu has had no 1.0 release, and its flags, labels, expressions, cookie names and log lines may change in any release, including a minor one, without a migration path.
> Read the [release notes](https://github.com/crazygolem/perdu/releases) before upgrading.

> [!NOTE]
> **Perdu isn't open source yet.**
>
> Its image and this manual are published, but its sources aren't: they will be with perdu's 1.0 release, likely under a copyleft licence.
