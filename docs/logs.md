# Reading the logs

Perdu logs to standard error, in the format [`--log-format`](flags.md#--log-format) names: `text`, a line of `key=value` pairs per event, or `json`, an object per line.
[`--log-level`](flags.md#--log-level) sets the lowest level logged, `info` by default.

Each line's message names its event and never varies, so a log collector can match an event by its message alone; everything that varies is an attribute beside it — the why in `reason`, an error's text in `error`.

The `text` format makes an effort to be readable, while the `json` format is meant to be parsed efficiently.
So the two can differ: in the `text` format the structure can be flattened, and one event can be logged on multiple lines (appearing as multiple events).
In the `text` format, for instance, a list reads as its elements in brackets, separated by commas, `[en,fr]`, an element quoted only when it holds an ambiguous character (comma, quote, bracket, brace) or is empty; an entity in a list reads as a record in braces.

## What the levels mean

* **`WARN`** is a problem with perdu's **inputs** — your labels, the request, a source of icons — that perdu worked around, and kept serving through.
  An operator watching `WARN` sees everything perdu chose to work around.
* **`ERROR`** is perdu's **own** operation failing.
  Every crash — a `panicked` line, or `panic` — is a defect of perdu's: its line holds the panic and where it happened, which is what a bug report needs.
* **`INFO`** is what perdu is doing, and the outcomes you expect but perdu can't read off your labels — including a **mismatch between two parties**, such as labels naming a placement the page doesn't show, where nothing tells perdu which side is wrong: it names both, and leaves the call to you.
* **`DEBUG`** is the same events where they are ordinary rather than worth a look.

A line about one of perdu's inputs identifies its source:
* **A line about a flag** names it and where its value came from — `flag`, `env` or `default` — as one attribute, `option`: in the text format the flag followed by its source in parentheses, `option=locales(default)`, and in JSON an object, `"option":{"name":"locales","source":"default"}`.
  A startup failure naming a second flag gives it as `with`, in the same form; the `configuration` lines, each about its flag, give it as `option` and where its value came from as `source`.
* **A line about a label names it**, by its key and the entity carrying it.
* **An entity** is named by its service's identity — the values of its `--identity-labels` — with its name and the full IDs of the containers defining it, and an action's key beside them.
* **A line about a request carries its client address**, as `client_address` ([Deploying perdu](deploy.md#the-clients-address)), so that you can act on who sent it.

**A value a rule reads or computes is logged by what happened to it** — its size, its outcome, its name — **never by its content**, since a rule can read a credential under a name perdu can't know; the groups on the `DEBUG` line of `audience tokens dropped` are the one exception.

**A problem with your labels is logged on every build while it lasts**, once per build for each label, so that it stays in recent logs.
**A source of icons failing is logged once** when it starts and once when it recovers, and the attempts in between at `DEBUG`.
**A refresh re-arms a failure logged once**: its first failure after the refresh is a `WARN` again, even within an outage already announced, and the later ones are at `DEBUG` as before ([Deploying perdu](deploy.md#refreshing)).
**A line about a failure perdu retries gives the longest wait before the retry, as `backoff`**: `build failed` always does, the others only when the level is `debug`, the only level at which their retries are logged.
After the CDN sends a `Retry-After`, `icon source failing` gives that wait as `pause` instead.
**A line about a request is logged every time**, never collapsed; its frequency is what discriminates one viewer from a client probing perdu: a client can therefore make as many of these lines as it sends requests ([Per-viewer data](rules.md#the-input-cap)).

## Startup

The startup events are emitted during the [startup sequence](deploy.md#starting).

| Message                                         | Level   | When                                                                                                                                                                                                                                                                                                                                                                     |
| ----------------------------------------------- | ------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `perdu starting`                                | `INFO`  | first, with perdu's `version`, so that a failed start carries it too                                                                                                                                                                                                                                                                                                     |
| `unknown PERDU_ variables ignored`              | `WARN`  | an environment variable starting with `PERDU_` that no flag reads, all of them named, in alphabetical order                                                                                                                                                                                                                                                              |
| `configuration`                                 | `INFO`  | every flag's value and where it came from, an unset flag's value empty: in the text format, a line per flag, and one per `--data` rule, with its `rule` and `expr`; in JSON, one record, `--data` a list of its rules, empty when there is none; an unset `--audience-expr` with a `note` saying whether filtering is off or comes from a `--data` rule named `audience` |
| `rule reads request attributes it doesn't name` | `WARN`  | a rule reaching a kind of request attribute without naming it, and not testing `oversized` for that kind ([Per-viewer data](rules.md#the-input-cap))                                                                                                                                                                                                                     |
| `listening`                                     | `INFO`  | each address perdu listens on, an empty host as `:<port>`                                                                                                                                                                                                                                                                                                                |
| `icons mount check failed`                      | `WARN`  | the `--icons-dir` directory not there yet, or unreadable, on the first attempt, with `backoff` when the level is `debug`; later attempts at `DEBUG`                                                                                                                                                                                                                      |
| `icons mount check slow`                        | `WARN`  | a read of the `--icons-dir` directory taking more than 5 seconds                                                                                                                                                                                                                                                                                                         |
| `icons mount empty`                             | `WARN`  | the `--icons-dir` directory empty, which usually means a mount that didn't land                                                                                                                                                                                                                                                                                          |
| `event stream connected`                        | `INFO`  | the docker event stream connecting for the first time ([The docker event stream](#the-docker-event-stream))                                                                                                                                                                                                                                                              |
| `startup complete`                              | `INFO`  | the first page served                                                                                                                                                                                                                                                                                                                                                    |
| `startup failed`                                | `ERROR` | perdu stopping before it serves: the flag at fault and where its value came from, or the `reason`, such as the startup window ending, with every step that hadn't succeeded and what it was waiting on or its last error ([Deploying perdu](deploy.md#starting))                                                                                                         |

The generated pages aren't logged at startup: they depend on the labels, which no build has listed yet ([Builds](#builds)).

## Builds

Each build logs `build starting`, saying why it ran, and then how it ended: `build completed` followed by `resource cache stats`, or a failure.

| Message                    | Level   | When                                                                                                                                                                        |
| -------------------------- | ------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `build starting`           | `INFO`  | a build beginning, with the kinds of `triggers` that asked for it                                                                                                           |
| `build completed`          | `INFO`  | a build's page served: its triggers, how long it took, and how many services, pages and icons it produced, each with what changed                                           |
| `resource cache stats`     | `INFO`  | after each `build completed`: the icons perdu holds                                                                                                                         |
| `build requests coalesced` | `DEBUG` | the debounce gathering several requests into one build ([How perdu works](how-it-works.md#when-a-build-runs))                                                               |
| `error pages rendered`     | `DEBUG` | error pages a build wrote, one per language perdu has chrome for, and how long they took together                                                                           |
| `build phase slow`         | `WARN`  | a phase of a build taking more than 10 seconds, naming it, with how many docker calls discovery made                                                                        |
| `build failed`             | `WARN`  | perdu failing to list the containers — docker unreachable or refusing — which perdu retries after at most `backoff`, growing to 30 seconds, while it serves the last page   |
| `build failed`             | `ERROR` | rendering the page failing, a defect of perdu's: the last page keeps being served, and the next build tries again; before the first page, perdu exits ([Startup](#startup)) |
| `build panicked`           | `ERROR` | a build crashing, with where: the last page keeps being served; before the first page, perdu exits ([Startup](#startup))                                                    |

**The triggers** are `startup`, `docker-event` — a container created, updated or removed — `reconnect` — the docker event stream coming back — `retry`, after a listing failed, `refresh`, after a `SIGHUP`, and `resource-change`, an icon the page uses having loaded or changed.
A burst of them, such as the events `docker compose up` sends, starts one build ([How perdu works](how-it-works.md#when-a-build-runs)).

**`build completed` answers "did my label edit land?"**
It counts `services`, `pages` and `icons`, each followed, in the text format, by what changed since the page it replaces, where anything did: `+` added, `-` removed, `~` differing.

```
level=INFO msg="build completed" triggers=[docker-event] duration=12.4ms services=14(~1) pages=5 icons=9 late=0
```

Here one service's labels changed, and nothing else.
A service counts as differing when what perdu made of its own labels did, not when its containers were recreated with the same labels, and a label adding a locale changes only the service carrying it.
A line with no change at all is a rebuild that produced the same page byte for byte.
The first build has nothing to compare against, so it carries no changes.
In JSON, each count is an object holding `count` and, from the second build on, `added`, `removed` and `changed`.
`late` counts the library icons the build had no answer for when it rendered, and which arrived before it ended: the next build shows them ([How perdu works](how-it-works.md#the-resource-cache)).

**`resource cache stats`** gives the icons perdu holds in its [resource cache](how-it-works.md#the-resource-cache): `entries`, with their bytes; `unused`, those the page doesn't use any more, which it keeps until a refresh; and `unanswered`, those still waiting for their library.

```
level=INFO msg="resource cache stats" entries=6(2773B) unused=2(665B) unanswered=0
```

## Labels and the page

Each of these is logged on every build while its cause lasts, once per build for each label, entity or locale it is about ([Labels](labels.md#what-happens-to-a-label-perdu-cant-use)).

| Message                                     | Level   | When                                                                                                                                                                                                                   |
| ------------------------------------------- | ------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `service excluded`, `action excluded`       | `WARN`  | an entity set aside, with the `reason`                                                                                                                                                                                 |
| `label ignored`                             | `WARN`  | a label perdu doesn't know under the prefix, or a locale it can't use                                                                                                                                                  |
| `unusable value`                            | `WARN`  | a `url`, an icon or a `data:` URI perdu can't use, with the `error` saying why, and the pages left without it                                                                                                          |
| `icon unresolved`                           | `WARN`  | an icon reference no library answers, with the answer, the source that gave it, and for a refusal its cause ([Icons](icons.md#when-an-icon-is-missing))                                                                |
| `chrome string missing`                     | `WARN`  | a chrome string missing from a page's language, shown as its key instead, which the built-in catalogs never do: seeing it means a bug in perdu                                                                         |
| `container not considered`                  | `INFO`  | a container carrying `enable` but missing an identity label, so never on the page                                                                                                                                      |
| `containers passed over`                    | `INFO`  | containers of the rank defining a service whose labels differ from those defining it, with whether the service is on the page; at `DEBUG` where the page doesn't depend on them ([Labels](labels.md#service-identity)) |
| `service left off`                          | `INFO`  | a service whose live container no longer carries `enable` ([Labels](labels.md#service-identity))                                                                                                                       |
| `placement not picked`                      | `INFO`  | entities placed at a `~` placement the page doesn't show, naming the placements and the entities                                                                                                                       |
| `audience declared with no rule configured` | `INFO`  | entities declaring an `audience` while no rule computes the viewer's groups                                                                                                                                            |
| `listed locale has no page`                 | `INFO`  | a tag `--locales` lists that no label or chrome asks for                                                                                                                                                               |
| `locale generated`                          | `DEBUG` | each page's locale, the token of `--locales` it expands from, if any, as `from`, what asked for it, and whether its chrome is the default locale's                                                                     |
| `locale not listed`                         | `DEBUG` | a label asking for a locale `--locales` doesn't admit                                                                                                                                                                  |

## Icons

Icon events are logged as perdu loads icons from its libraries, in the background ([How perdu works](how-it-works.md#the-resource-cache)), and can happen outside of builds.

| Message                                    | Level   | When                                                                                                                                                                                                                                        |
| ------------------------------------------ | ------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `icon source failing`                      | `WARN`  | a source of icons — the `--icons-dir` directory, or the CDN — starting to fail, once, naming its pipeline as `pipeline`, the cause and the icon that hit it, with `backoff` or `pause` when the level is `debug`; later failures at `DEBUG` |
| `icon source answering again`              | `INFO`  | that source answering again                                                                                                                                                                                                                 |
| `icon reference failing`                   | `WARN`  | one icon's own file failing to read, once, with `backoff` when the level is `debug`; later failures at `DEBUG`                                                                                                                              |
| `icon reference loading again`             | `INFO`  | that icon loading again                                                                                                                                                                                                                     |
| `icon load rejected`                       | `WARN`  | an icon refused, with the cause: a `4xx`, a wrong type, too large, a path that isn't a file, or one leading out of the directory                                                                                                            |
| `icon mount read slow`                     | `WARN`  | a read of the `--icons-dir` directory taking more than 5 seconds, once per stuck read                                                                                                                                                       |
| `icon load declined`                       | `WARN`  | a library declining to load an icon, which the built-in ones never do                                                                                                                                                                       |
| `icon load accepted`, `icon load answered` | `DEBUG` | each icon a library starts loading, and each answer, naming the source that gave it                                                                                                                                                         |

## Requests

| Message                                  | Level   | When                                                                                                                                                                                                 |
| ---------------------------------------- | ------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `request attribute over the input cap`   | `WARN`  | a header, query parameter or cookie over 8 KiB that a rule reads without testing `oversized`, naming the rules computing from `""`; otherwise at `DEBUG` ([Per-viewer data](rules.md#the-input-cap)) |
| `rule evaluation failed`                 | `WARN`  | a rule failing on a request, with what failed                                                                                                                                                        |
| `audience tokens dropped`                | `WARN`  | groups `--audience-expr` dropped, by count                                                                                                                                                           |
| `audience tokens dropped`                | `DEBUG` | the same groups, by value, on a line of its own after the `WARN`                                                                                                                                     |
| `rule evaluated`                         | `DEBUG` | every rule's outcome on every page request: a value and its size, `nil`, or `failed`                                                                                                                 |
| `dashboard page served with no rule run` | `DEBUG` | a page request that isn't a navigation, saying whether the browser marked it as a subresource, such as an image, or it carried no mark at all                                                        |
| `cookie failed to decode`                | `DEBUG` | a cookie of perdu's that it can't decode, which counts as absent                                                                                                                                     |

## The docker event stream

| Message                    | Level  | When                                                                                                                                                       |
| -------------------------- | ------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `event stream connected`   | `INFO` | the stream connecting: the first time, after a socket proxy closed an idle connection or a refresh reopened it, and after an outage, so that its end shows |
| `event stream open failed` | `WARN` | opening the stream failing, the first time in an outage, with `backoff` when the level is `debug`; later attempts at `DEBUG`                               |
| `event stream ended early` | `WARN` | a stream ending soon after it opened, the first time, with `backoff` when the level is `debug`; later ones at `DEBUG`                                      |

While the stream is down, perdu keeps serving its last page and reconnects, backing off to an attempt every 30 seconds; the reconnect lists every container again, so nothing missed meanwhile is lost.

## Signals

| Message              | Level  | When                                                                                                                                          |
| -------------------- | ------ | --------------------------------------------------------------------------------------------------------------------------------------------- |
| `refresh beginning`  | `INFO` | `SIGHUP` received, naming it: the refresh has begun, and its result shows on the build that follows ([Deploying perdu](deploy.md#refreshing)) |
| `shutdown beginning` | `INFO` | `SIGTERM` or `SIGINT` received, naming it ([Deploying perdu](deploy.md#stopping))                                                             |

## Other errors

| Message                      | Level   | When                                                                                                                                          |
| ---------------------------- | ------- | --------------------------------------------------------------------------------------------------------------------------------------------- |
| `listener stopped accepting` | `ERROR` | a listening address failing, after which perdu exits                                                                                          |
| `http server error`          | `ERROR` | an error of the HTTP server's own, such as a failing accept it retries or a misused response, with the server's text in `error`               |
| `handler panicked`           | `ERROR` | a request crashing, with the request's client address (your proxy's, before reading `X-Forwarded-For`): that request fails, and perdu goes on |
| `icon load panicked`         | `ERROR` | loading an icon crashing: that icon is refused, and perdu goes on                                                                             |
| `panic`                      | `ERROR` | one of perdu's own components crashing, after which perdu exits                                                                               |
