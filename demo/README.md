# perdu's demo

perdu showing the dashboard of a family's home server: films and music, documents, the house, a few tools, and an administration section, in English, French and German, behind a login.

The services are stand-ins, containers that carry the labels and run nothing else, so their links lead to addresses under `home.example`, which never resolve.
Log out is the one link that works.

## Running it

It needs Docker with Compose, or podman with podman-compose, and the network: for the images, and for the services' icons, which perdu fetches from jsDelivr.

From this directory:

```sh
docker compose up -d
```

Under podman, name its socket:

```sh
DOCKER_SOCKET="$XDG_RUNTIME_DIR/podman/podman.sock" podman-compose up -d
```

Then open <http://localhost:8080/>, and stop it with `docker compose down`, or `podman-compose down`.

`PERDU_IMAGE` runs another image of perdu than the published one, such as one built from its source.

The demo's labels are under `perdu-demo.` rather than `perdu.`, which perdu reads under `--namespace=perdu-demo`, so that a perdu already running on the same daemon shows none of the demo's tiles, and the demo none of its services.
A stack of your own writes `perdu.name` where `compose.yaml` writes `perdu-demo.name`.

## Logging in

The front end stands in for a login: a real deployment's proxy signs viewers in with an identity provider and tells perdu their groups and their name, in headers that `--audience-expr` and `--data` read.
Here, the login page offers three viewers, and the front end sets the headers for the one picked:

| Login | Groups         | Sees                                                           |
| ----- | -------------- | -------------------------------------------------------------- |
| Guest | `guest`        | the media, and Log out                                         |
| Sam   | `users`        | everything but Forgejo and the administration section, greeted |
| Alex  | `users,admins` | everything, greeted                                            |

Log out goes back to the login page.

## What to look at

* **The languages.** The switcher shows the page in English, French and German.
  Most services are described in French, but only Jellyfin and Grandma's photos in German: elsewhere, the German page shows the English description, marked as English for browsers and screen readers.
  Forgejo is described in English alone.
* **Grandma's photos** link nowhere, so the tile is dimmed, and their description is too long for the tile: hover over it, focus it with the keyboard, or click it to read it whole.
  Without an icon, the tile shows a plate with its initials.
* **Nextcloud's actions** are listed in its tile.
  authentik's actions, Change password and Log out, are placed in a section authentik isn't in, so each gets a tile of its own there, with authentik's name as its second line, and is shown to the viewers its own `audience` names: Change password to users, Log out to everyone.
* **The administration section** comes last, shown to Alex alone.
  So is Forgejo, though it stays among the tools: a section places a tile, and its `audience` alone says who sees it.
* **The icons** come from dashboard-icons, which perdu fetches and serves itself, but for Navidrome's and BookStack's, given as URLs of selfh.st's icons, which the viewer's browser fetches.
* **Your system's settings**: the page follows its dark or light scheme, more contrast, forced colours and reduced motion.
