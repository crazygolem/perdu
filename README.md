<div align="center">
  <img src="media/logo/icon-sba.svg" alt="" width="144" height="64">
</div>

# Perdu

Perdu is a landing page for self-hosted application stacks.

It reads labels from your docker containers, builds a static page per language listing the services they describe, and serves those pages behind your reverse proxy, which tells perdu who is viewing, so that each viewer's page shows the services meant for them.

![The demo's page, listing a home server's services with their icons and descriptions, split diagonally between its light and dark schemes](media/screenshot.png)

I'd love to hear your feedback if you try perdu; don't hesitate to open a [discussion](https://github.com/crazygolem/perdu/discussions) on GitHub.

* **The manual:** <https://crazygolem.github.io/perdu/> for the latest release, or [`docs/`](docs/README.md) here in the repository under each release's tag.
* **The docker image:** `ghcr.io/crazygolem/perdu`, for `linux/amd64` and `linux/arm64`.
* **The demo:** [`demo/`](demo/README.md), a family's home server behind a login that offers three viewers.
* **The release notes:** [`CHANGELOG.md`](CHANGELOG.md), and each release on the [Releases](https://github.com/crazygolem/perdu/releases) page.

> [!IMPORTANT]
> Perdu has had no 1.0 release, and nothing in it is a compatibility promise yet: its flags, labels, expressions, cookie names and log lines may change in any release, including a minor one.

## Running it

```sh
docker pull ghcr.io/crazygolem/perdu:latest
```

To try it on the machine running Docker, give it the Docker socket, and the locale its pages fall back to, the one flag it requires:

```sh
docker run --rm -p 127.0.0.1:8080:8080 \
    -v /var/run/docker.sock:/var/run/docker.sock \
    --group-add "$(stat -c %g /var/run/docker.sock)" \
    ghcr.io/crazygolem/perdu:latest \
        --default-locale=en \
        --identity-labels=''
```

Then open <http://localhost:8080/>, which lists the containers carrying perdu's [labels](docs/labels.md): none until you add some.
This one runs nothing and carries a service's labels:

```sh
docker run -d --name perdu-example --network none \
    --label perdu.enable=true \
    --label perdu.name=Example \
    --label perdu.description='An idle container, labelled for perdu' \
    --label perdu.url=https://example.com/ \
    --label perdu.icon=di:docker \
    docker.io/alpine:3.24 tail -f /dev/null
```

Reload the page to see its tile, and remove it with `docker rm -f perdu-example`.

By default, perdu expects to run in a compose stack; the `--identity-labels=''` option makes perdu read its labels from each container individually instead.

> [!NOTE]
> Mounting the socket is root-equivalent on the host, which suits a try but not a deployment: [Deploying perdu](docs/deploy.md) has a compose stack snippet reaching the docker API through a socket proxy, and describes what the reverse proxy in front of perdu must do.

## This repository

Perdu's sources aren't public yet: they will be with perdu's 1.0 release, likely under a copyleft licence.

This repository holds what its releases publish, the manual, the demo, the release notes and the licences, and is updated automatically.
Changes made here don't survive the next update.

## Licences

* Perdu's image, its binary and what ships with it, is licensed under the MIT licence, in [`LICENSE`](LICENSE), as is the rest of this repository where nothing below says otherwise.
  [`NOTICES`](NOTICES) gives the licences of what the image ships besides perdu, which it also carries in `/licenses/`.
* The manual, [`docs/`](docs/), is licensed under CC BY-SA 4.0, and its examples under 0BSD: [`docs/LICENSE`](docs/LICENSE).
* The demo, [`demo/`](demo/), is licensed under 0BSD: [`demo/LICENSE`](demo/LICENSE).
* Perdu's icon and favicon, in [`media/logo/`](media/logo/) and copied in `overrides/` for the site, aren't covered by those licences: all rights are reserved, except redistributing them unmodified as part of perdu and using them unmodified to refer to perdu, as [`media/logo/LICENSE`](media/logo/LICENSE) says.
* The screenshots aren't covered by those licences: the logos they show belong to their owners.
