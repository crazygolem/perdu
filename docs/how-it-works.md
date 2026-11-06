# How perdu works

This chapter explains what perdu is made of and how data moves through it: from your containers' labels to the pages, and from an icon's reference to the page, through a build, a refresh and a request.
It explains rather than lists: the other chapters hold the rules and the values, and link here for the mechanisms they share.

## Three fronts

Perdu works on three fronts, which meet in the pages:

* **Builds** generate the pages before any viewer requests them: perdu lists your containers, reads their labels and renders a page per language, which it keeps in memory.
* **Requests** are answered from the latest build's pages, as the build wrote them: every viewer gets the same page for a language, and what differs between viewers reaches the page in cookies, which its scripts read in the viewer's browser.
* **Pages** run scripts in the viewer's browser that customize the page's look and enable dynamic features.

```
      docker events, a refresh,                                           BUILDS
      an icon having loaded
                │
                ▼
      build coordinator: decides when a build runs
                │
                ▼
      a build:  • discovery      ◄── docker: the containers and their labels
                • configuration
                • rendering      ◄── resource cache ◄── your icons directory,
                │                                       Font Awesome, the CDN
                ▼
      bundle: the pages, and the assets and icons they use
                │
                ▼
      HTTP server ──► your rules ──────► cookies                        REQUESTS
        ▲       │                           │
request │       │ the page                  │ on the response
        │       ▼                           ▼
      the viewer's browser: the page, whose scripts read the cookies       PAGES
```

## Perdu's parts

| Part                  | What it does                                                                                                                                     |
| --------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------ |
| The event stream      | perdu's connection to docker's events: it tells the build coordinator when a container carrying `<prefix>.enable` is created, updated or removed |
| The build coordinator | decides when a build runs, and runs one at a time                                                                                                |
| Discovery             | a build's first phase: lists the containers carrying `<prefix>.enable`, and groups them into services                                            |
| Configuration         | a build's second phase: reads the services' labels into what each page shows                                                                     |
| Rendering             | a build's third phase: writes each page from the built-in template, taking the icons it needs from the resource cache                            |
| The resource cache    | holds the icons perdu has loaded from the icon libraries, and loads new ones in the background                                                   |
| The HTTP server       | answers each request from the latest build, and runs your rules when a viewer opens a page                                                       |
| Your rules            | your `--data` and `--audience-expr` expressions, which compute values for each viewer from their request                                         |
| The page's scripts    | run in the viewer's browser: they read your rules' values, hide the tiles not meant for the viewer, and add touches such as the greeting         |

The logs name a build's phases as this table does.
Everything but the page's scripts runs in perdu's one process.

## Builds

### When a build runs

A build runs whenever something may have changed what the page shows, a **trigger** as the logs call it:

* at startup, once perdu has reached docker and, with `--icons-dir` set, found your icons directory ([Deploying perdu](deploy.md#starting));
* when a container carrying `<prefix>.enable` is created, updated or removed, as docker's event stream tells perdu;
* when the event stream connects again after it dropped, since the events sent meanwhile are lost;
* after a listing failed, to try it again;
* after a refresh ([Refreshes](#refreshes));
* when an icon the page uses has loaded or changed ([The resource cache](#the-resource-cache)).

Starting or stopping a container runs none ([Labels](labels.md#service-identity)).

**Triggers are debounced**: a build starts half a second after the last trigger, and at most 5 seconds after the first, so the burst of events `docker compose up` sends makes one build, not one per container.
Until the first page is served, builds aren't debounced, so that perdu serves its first page as soon as it can ([Deploying perdu](deploy.md#starting)).

**One build runs at a time.**
Triggers arriving while one runs make one more build after it, however many arrive.
The log line starting a build names the kinds of trigger it covers ([Reading the logs](logs.md#builds)).

**An event is a signal, not data.**
Every build lists the containers afresh and builds from that listing alone, never from what an event said.
So no build depends on an event having reached perdu: one lost while the event stream was down is made up for by the next listing.
Unless a failure occurs, **builds are always run to completion**: perdu might get inputs identical to the previous build's, different inputs that lead to the same result, or discard the new inputs; none abort the build, it simply produces the same pages, byte for byte.

### What a build does

1. **Discovery** asks docker for the containers carrying `<prefix>.enable`, stopped ones included, groups them into services by their identity labels, and picks the containers that define a service ([Labels](labels.md#service-identity)).
   It is the only phase that reads anything from outside perdu, and it reads only from docker.
   When docker can't be reached, the build fails here, and perdu tries again, with a progressive backoff of up to 30 seconds, while it serves the last pages it built, or, before the first, until the startup window ends ([Deploying perdu](deploy.md#starting)).
2. **Configuration** reads each service's labels: it sets aside the entities whose labels it can't use, decides which languages get a page ([Locales](locales.md#which-pages-exist)), and chooses each page's value for each field, its own language's or the one it falls back to ([Locales](locales.md#falling-back)).
   It also collects the audiences your labels declare, which your rules can read ([Per-viewer data](rules.md#writing-an-expression)).
3. **Rendering** writes each page from the built-in template, checking each value it uses, such as a `url`'s shape or an icon's form, and taking each library icon from the resource cache as the cache holds it at that moment ([From an icon's reference to the page](#from-an-icons-reference-to-the-page)).
   Separately, a missing-locale error page is generated for each language perdu has chrome for, used when the viewer requests a locale that has not been otherwise generated and has no appropriate fallback page.
   A failure here is a defect of perdu's, which a retry would only repeat, so perdu doesn't retry on its own: the last pages stay served until a later build, whatever triggers it, succeeds, and before the first page perdu stops instead.

> [!NOTE]
> **Perdu never blindly lists or reads containers**: it lists those carrying `<prefix>.enable` and those sharing a stopped service's identity; the label's value itself belongs to discovery:
>
> 1. List the containers that carry `enable`;
> 2. Group them into services by their identity labels, and pick the containers that define a service — the others are ignored;
> 3. Read `enable`'s value from those defining containers: unparseable and conflicting values set the service aside;
> 4. If a service is defined only by stopped containers: make a targeted check for the existence of a live container with the same identity but without the `enable` label: if there is one, the service is ignored.
>
> Perdu reads the other labels only after a service's identity has been established, from the containers that define it ([Labels](labels.md#service-identity)).

A label perdu can't use costs only the entity or the value carrying it, never the build, and is logged on every build until you fix it ([Labels](labels.md#what-happens-to-a-label-perdu-cant-use)).

**A build's result is a bundle**: the pages and missing-locale error pages, their stylesheet, scripts and favicon, and the icons they use, with what requests need from the build — the languages that have a page, and the audiences your labels declare.
**The new bundle replaces the old one at once**, so a request is answered from one bundle or the other, never from parts of both.
It lives in memory alone, and a restart builds it again from a fresh listing.

Each page and asset carries an `ETag` computed from its content, so one a build didn't change keeps its `ETag`, and a browser revalidating it gets a cheap `304` ([Deploying perdu](deploy.md#caching)).

## From an icon's reference to the page

An icon comes in one of four forms ([Icons](icons.md#the-forms-an-icon-takes)), and only one goes through the resource cache:

* **An `http` or `https` URL, or a root-relative path**, goes into the page as it is, and the viewer's browser fetches it: perdu never does.
* **A `data:` URI** is decoded by the build itself, and served from `/assets/icons/~data/`.
* **A library icon**, `<library>:<name>` such as `di:navidrome`, is a reference the **resource cache** answers.

Rendering checks a library icon's form, then takes the cache's answer for it, whatever the cache holds at that moment:

* **the icon**: the bundle gets it, and the page links it from `/assets/icons/<library>/<name>.<extension>`;
* **not found, or refused**: the page draws the default in its place, the service's initials on a plate or an action's bolt, and the build logs a `WARN` ([Icons](icons.md#when-an-icon-is-missing));
* **no answer yet**, for an icon the cache is still loading or has never held: the page draws the default for now, and the cache loads the icon in the background.

Once the icon has loaded, the cache triggers a build: the icon appears with the following build, usually within a second, so a page loaded in between shows the default.

In the viewer's browser, the page loads the icon as an image, and a script draws it in the page's text colour when it is monochrome ([Icons](icons.md#how-the-page-draws-icons)).

## The resource cache

**The resource cache holds the icons perdu has loaded from its icon libraries**, so that a build never waits for a file or the network, and perdu doesn't read or fetch again an icon it already holds.

Every library icon goes through it, whatever its library: `fa`, `di`, and the libraries you add in your icons directory, the files there overriding `fa`'s and `di`'s icons included, and so does the bolt the page draws for an action without an icon, which is Font Awesome's `fa:solid-bolt`.
No other form of icon enters it ([From an icon's reference to the page](#from-an-icons-reference-to-the-page)), and neither do the stylesheet, the scripts and the favicon, which are built into perdu.

**It holds one answer per icon**, however many services, pages and languages use it: the icon itself, *not found*, or *refused* with its cause, such as a file over the size limit or a CDN's `403`; or no answer yet, while the icon loads.

**A build takes what the cache holds, and never waits.**
The first answer a build gets for an icon stands for the rest of that build, so every page of a build draws an icon the same way.
An icon the cache doesn't hold yet is loaded in the background, while the build goes on without it.

**Icons load through a pipeline per library**, a chain of sources each tried in turn:

| Pipeline  | Libraries           | Sources, in order                                     |
| --------- | ------------------- | ----------------------------------------------------- |
| `fa`      | `fa`                | your icons directory, then the icons built into perdu |
| `di`      | `di`                | your icons directory, then jsDelivr's CDN             |
| `default` | every other library | your icons directory                                  |

**Your icons directory is in each chain when `--icons-dir` is set** ([Icons](icons.md#your-own-icons)).
A source that doesn't have the icon passes it on to the next, and after the last one the answer is *not found*.
An answer that needs no file and no network is there on the spot, so the build asking for it already has it: **without an icons directory**, `fa`'s icons come from the built-in set at once, and the libraries other than `fa` and `di` are *not found* at once.

**A new answer for an icon the page uses triggers a build**, and so does one arriving for an icon the build in progress used, once that build ends.
That's how an icon perdu had to read or fetch appears on the build after the one that first used it.
Answers arriving close together are debounced, as docker events are ([When a build runs](#when-a-build-runs)), and an answer identical to the one already cached doesn't trigger a build.

**Answers stay until a refresh.**
Docker is the only source that tells perdu of its changes: perdu doesn't watch your icons directory ([Icons](icons.md#your-own-icons)), and doesn't ask the CDN again for an icon it holds.
So a build doesn't reach out to the CDN for the icons perdu already holds, and isn't slowed when the CDN is.
In return, a change to a file or to an icon upstream shows only once a refresh reloads it ([Refreshes](#refreshes)), and so does a fix to an icon perdu found missing or refused: until then, the page shows what perdu holds.

**Icons the page no longer uses are kept until a refresh**, so that a service that goes away for a while finds its icons again when it comes back, without a read or a fetch.
The first build **after a refresh** clears the icons it doesn't use, and nothing else does: until then, the cache keeps every library icon a build has used.
The logs give what it holds after each build, and how much of it the page no longer uses ([Reading the logs](logs.md#builds)).

**A failure is not an answer.**
A source that fails — your icons directory missing or unreadable, a file perdu can't read, the CDN unreachable, too slow, or failing on its side — leaves the icon's answer as it was, and perdu tries again later, with a progressive backoff of up to 30 seconds for your icons directory, and 10 minutes for the CDN.
The backoffs are kept per source, so an outage costs about one attempt per wait rather than one per icon, and per icon, so one broken icon doesn't slow down the others.
An outage is logged when it starts and when it ends, rather than on every attempt ([Reading the logs](logs.md#icons)).
**Perdu never gives up on an icon**: one whose source is down stays without an answer until the source recovers, while the icons already loaded keep showing.

**A pipeline works through its icons one at a time at each source**, from a queue per source: `fa`, `di` and `default` each read your icons directory through a queue of their own, the `default` one serving every library you add.
So a read that hangs holds every icon queued behind it in that pipeline, and a directory that stops answering, as a network share does when its server hangs, holds icons in all three ([Icons](icons.md#your-own-icons)).

**A restart starts with an empty cache**, which lives in memory: every icon is loaded again.
With `--icons-dir` set, every library icon is read from the directory first, Font Awesome's included, so the first page after a restart shows none of them, not even the bolt, and the build after it does.

## Refreshes

A refresh, which `SIGHUP` triggers ([Deploying perdu](deploy.md#refreshing)), catches up with what perdu can't learn on its own: changes to your icons directory and to upstream icons, and a docker event stream that has silently stopped delivering.
Perdu:

1. cuts short the wait before your icons directory's next check, if startup is still waiting for it;
2. reopens the docker event stream;
3. reloads every icon the served pages use, from its source, **forgetting the backoffs that failures had built up**, so that a source you have just fixed is tried at once.
   Each icon keeps its answer until the new one arrives, so a source still down costs nothing;
4. builds from a fresh listing once the reopened stream connects, as after any reconnection: that build **drops the icons it doesn't use from the cache**.

An icon whose reload brings a different answer triggers a build, as an answer for a new icon would ([The resource cache](#the-resource-cache)).

## Requests

Every request is answered from the bundle served when it arrives, and from that bundle alone ([Deploying perdu](deploy.md#paths) lists the paths):

* **`/`** redirects to the page in the viewer's language, chosen among the bundle's pages from the choice the viewer stored and the languages their browser requests ([Locales](locales.md#which-page-a-viewer-lands-on)).
* **A page**, `/l/<locale>/`, is sent as the build wrote it.
  When the request is the viewer opening the page, perdu also runs your rules over the request, and sets each result as a cookie on the response, a `304` included ([Per-viewer data](rules.md)).
  A request for a language with no page redirects to the closest page there is, or gets the missing-locale error page where none is close enough, with a `404` and no rule run ([Locales](locales.md#a-locale-with-no-page)).
* **`/set-lang/<locale>`** stores the viewer's choice of language in a cookie, and redirects to its address, `/l/<locale>/`.
* **`/assets/`** serves the stylesheet, the scripts, the favicon and the icons the pages use; the page can also embed the assets it uses, and in this release it embeds all its scripts.

Until the first page is served, every request gets a `503` ([Deploying perdu](deploy.md#starting)).


## Pages

**In the viewer's browser, the page customizes itself.**
Its first script reads the cookies your rules set; another hides the tiles not meant for the viewer before they are drawn; others fill in the greeting and add the page's presentational touches ([The template](template.md)).

Each cookie also carries a short hash of the audiences its value was computed against, which the page compares with its own build's, warning in the browser's console when they differ ([Per-viewer data](rules.md#the-environment-identity)).

Without JavaScript, or without a value for the viewer's groups, nothing is hidden: the page fails open ([The template](template.md#without-javascript)).
