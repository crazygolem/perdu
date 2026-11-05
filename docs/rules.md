# Per-viewer data

Perdu builds its pages once, the same for every viewer.

Whatever differs between viewers reaches a page another way: on each page request, perdu runs expressions you write over the request's headers, and sets their results as cookies beside the page, which the page's scripts read before they show anything.

The built-in template uses two such values: the viewer's groups, to hide the tiles not meant for them, and their name, to greet them.

## Rules

Each value comes from a **rule**: a name and an expression.

* `--data '<name>=<expr>'` defines one, and can be given once per name.
* `--audience-expr '<expr>'` defines the rule named `audience`, which perdu checks and cleans for the page's filtering ([The viewer's groups](#the-viewers-groups)).

```
--audience-expr 'headers["remote-groups"] | split(",")'
--data 'name=headers["remote-name"]'
```

**Rules run on page requests, and only there.**
A successful request for a page, `/l/<locale>/`, runs every rule when it is a navigation — the viewer opening the page, top-level or in a frame, its revalidation answered `304` included — and sets each result as a cookie on the response.
Nothing else runs them: not `/`, not `/set-lang/`, not `/assets/`, not a page address that answers with an error or redirects, and not a page the browser fetches as an image or a script, so that an icon pointing at the page can't make every viewer's browser run your rules.
A client that isn't a browser, such as `curl`, sends none of the marks of a navigation, and gets the page with no rule run and no cookie set ([Debugging a rule](#debugging-a-rule)).

**Each value travels in a cookie named `<prefix>_<name>`**, `perdu_audience` and `perdu_name` under the default namespace: a session cookie, scoped to the base path, withheld from a page framed under another site (`SameSite=Lax`), marked `Secure` as determined by [`--cookie-secure`](flags.md#--cookie-secure), and readable by the page's scripts (`HttpOnly` not set).
Its value is encoded in a perdu-internal format, which can change without notice: nothing but perdu should read it.

### Every value a rule computes is published to the page

A rule's cookie can't be kept from scripts, since the page's scripts read it: any script on perdu's origin can read it, and so can any application sharing its host's cookie jar ([Deploying perdu](deploy.md#sharing-a-host-name-with-other-applications)).
It is kept for the session, and sent with every request under the base path.

**So derive from a credential, never carry one.**
A rule decoding a user's email address out of a token your proxy forwards publishes the email, which is what it was written to do; on the other hand `--data 'token=headers["x-forwarded-access-token"]'` publishes the whole token to every script on the host.
Perdu can't tell the two apart, so writing safe rules is your responsibility.

Perdu keeps the request's `Authorization` and `Proxy-Authorization` headers out of what an expression can read, as basic hygiene rather than a protection: a token your proxy forwards under another name reaches a rule like any other header.
If a rule needs what those headers carry, have your proxy pass it on in headers of its own, such as `Remote-User`, or forward the token under another name and derive from it, as the `user` example does with `x-forwarded-access-token` ([Writing an expression](#writing-an-expression)).

### The environment identity

Each value's cookie also carries the **environment identity** it was computed against: a short hash of what your rules read from the build rather than from the request, which today is only `audiences`, the set of audiences your labels declare.

The page holds its own build's identity, and its scripts warn in the browser's console when a cookie's identity differs, then use the value anyway ([Debugging a rule](#debugging-a-rule)).

## Writing an expression

Expressions are written in [expr](https://expr-lang.org)'s language, and read the request through these names:

| Name        | Holds                                                                                                                                                                                                                                                                   |
| ----------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `headers`   | the request's headers, by name in **lower case** — `headers["remote-groups"]`, never `headers["Remote-Groups"]` — a header sent several times joined with `,` and no space; `host` among them, and `cookie`, `authorization` and `proxy-authorization` left out         |
| `query`     | the query's parameters, by name exactly as sent, its case included, a parameter given several times joined with `,`                                                                                                                                                     |
| `cookies`   | the request's cookies: a rule's own under the rule's name — `cookies["audience"]`, not `cookies["perdu_audience"]` — and the language cookie as `cookies["lang"]`, decoded; any other under its own name, as sent, unless a rule or the language cookie takes that name |
| `audiences` | the set of every `audience` value the labels of the current page's services and actions declare; test it with `in`: `"admins" in audiences`                                                                                                                             |
| `oversized` | which request attributes were too large to read on this request: `oversized.headers`, `oversized.query` and `oversized.cookies`, each a set of names ([The input cap](#the-input-cap))                                                                                  |

* A name the request doesn't hold reads as `""`, in all three maps: no `?.` or `??` is needed for them.
* Brackets always work, `headers["remote-groups"]`; a dot works only for a name that is an identifier, `query.page`, so prefer brackets for headers, whose names often hold hyphens.
* `split` is a function, not a method: `split(headers["x"], ",")`, or `headers["x"] | split(",")`; `headers["x"].split(",")` doesn't compile.
* `headers` holds what Go's HTTP server makes of the request, so a header it keeps out of its own map, such as `Transfer-Encoding`, isn't there.
* **`cookies` holds what the browser sent**, so a rule reading another rule's cookie reads the value that rule computed for the viewer's previous page — `""` on the first page of a session — not the one it computes for this one.
  A rule that needs the current value reads the headers that value comes from.

Two more examples, using expr's functions:

```
--data 'debug=query.myparam == "" ? false : fromJSON(fromBase64(replace(query.myparam, " ", "+"))).debug ?? false'
--data 'user=headers["x-forwarded-access-token"] == "" ? nil : (
    let s = split(headers["x-forwarded-access-token"], ".")[1] | replace("-", "+") | replace("_", "/");
    fromJSON(fromBase64(s + repeat("=", (4 - len(s) % 4) % 4))).email)'
```

### What a rule gives

An expression's result becomes its cookie's value by its type:

| Result                            | The value                                                                     |
| --------------------------------- | ----------------------------------------------------------------------------- |
| a string                          | itself                                                                        |
| a boolean                         | `true` or `false`                                                             |
| a number                          | its decimal form, never in exponent form; `+Inf`, `-Inf` and `NaN` as spelled |
| `nil`                             | **no value**                                                                  |
| a list, a map, or *anything else* | a failure, which gives **no value**                                           |

A list fails rather than being joined, since perdu can't know what separator the value's reader wants: join it yourself, as in `join(groups, ",")`.
A number with a unit of its own, such as a duration, counts as *anything else*, and so does a string that isn't valid UTF-8.

**`nil` is how an expression says it can't tell.**
It gives **no value**, which the page reads as "unknown" rather than as empty, and so fails open.
`headers["remote-groups"] == "" ? nil : headers["remote-groups"]` shows every tile to a request that came round your proxy without its header, where leaving the test out would treat it as a viewer with no groups.
A rule can also give `nil` without meaning to, from a missing field of a `fromJSON()` value; perdu can't tell the two apart, and logs every rule's outcome at `DEBUG`.

**A rule that fails gives no value** too, and logs a `WARN`, naming the rule and what failed — a result of the wrong type, a value over the output limit, or a runtime error, by its position in the expression alone.
The engine's own message isn't logged, since it can quote the request data the rule failed on.

A rule giving no value still sets its cookie, holding no value, so that the viewer's previous value doesn't linger.

### The output limit

**A value over 1 KiB is a failure**, counted in bytes on the computed value, which is encoded in UTF-8.
expr's `len()` counts characters rather than bytes, so the two agree only on ASCII: 400 Japanese characters pass `len(v) < 1024`, and fail the 1 KiB limit at about 1.2 KiB.
Keep a value well under the limit rather than measuring it.

The cookie carrying a value can be up to about three times its length, once encoded, so perdu's cookies come to at most about 3 KiB per rule, plus the language cookie.
That matters where something on the way limits header sizes, a response's especially: nginx reads a response's headers into one buffer, 4 or 8 KiB by default, and answers `502` for a response whose headers don't fit, so several large values can cost the whole page rather than one cookie.

**A browser silently drops a cookie whose name and value together exceed about 4 KiB.**
A very long `--namespace` beside a large value that encodes heavily can have its cookie dropped, and the browser then keeps whatever value that cookie held before, which may be stale.

## The viewer's groups

`--audience-expr` computes the viewer's groups, which the page matches against each tile's [`audience`](labels.md#audience) label.
It is a rule named `audience` like any other — executed, limited, logged and failing the same way — with two differences: the results it accepts, and the cleaning perdu gives them.

| Result                                                                   | Becomes                                                  |
| ------------------------------------------------------------------------ | -------------------------------------------------------- |
| a string                                                                 | split at its commas into groups                          |
| a list of strings                                                        | its elements, each one group; an empty list is no groups |
| `nil`                                                                    | no value                                                 |
| anything else — a boolean, a number, a list holding anything but strings | a failure                                                |

**Perdu cleans the groups**: it trims each, drops empty ones and repeats silently, drops each one outside `audience`'s characters, `[a-zA-Z0-9._-]+`, and joins what is left with commas.
So `Remote-Groups: users, admins` gives `users` and `admins`, and an absent header, which `split` turns into one empty string, gives no groups and no warning.
A list element holding a comma is dropped, never split.
Dropped groups are logged at `WARN` by count only, since they are a person's group memberships; their values are at `DEBUG`.
A dropped group could never have matched a label's `audience` in the first place, so dropping it changes nothing the viewer sees.

The output limit applies to the cleaned, joined value.

**What the page does with it:**

* **No rule computes `audience`** — neither `--audience-expr` nor a `--data` rule named `audience` — so filtering is off: every viewer sees every tile, and each build whose labels declare an `audience` logs an `INFO` line naming them.
* **No value** — the rule gave `nil` or failed, or the page has no cookie to read, as when a browser restarted shows a page from its cache — so filtering fails open: nothing is hidden.
* **A value** — the viewer's groups: the page hides every tile whose `audience` names none of them.
  **An empty value means the viewer has no groups**, so only the tiles without an `audience` show.
  It comes from the rule itself, or from a header the rule reads that was too large to read ([The input cap](#the-input-cap)).

**Narrowing the groups.**
`audiences` lets a rule keep only the groups your labels use, which keeps the value small for a viewer belonging to many: `split(headers["remote-groups"], ",") | map(trim(#)) | filter(# in audiences)`.
It is one way among others, and the choice of what to keep is yours: a group the value drops is one nothing on the page can match, so keep every group the page acts on — `# in audiences or # == "oncall"` is as good as the bare filter.

`--data 'audience=…'` is also supported, and in contrast with `--audience-expr` gets no checking or cleaning: its value has to be what the page expects, groups joined with commas, each matching `[a-zA-Z0-9._-]+`, or the viewer sees only the tiles without an `audience`.
It can't be set together with `--audience-expr`.

Perdu ships no audience expression of its own: one would have to assume a header name your proxy may not use.

## The greeting

The page greets the viewer by name — "Welcome, Ada!" — from the value named **`name`**, so the greeting needs a rule of that name:

```
--data 'name=headers["remote-name"]'
```

Without such a rule, or while it gives no value or an empty one, the page shows no greeting at all, rather than a greeting with a hole in it.
The value is written into the page as text, so it can never inject markup.

## The input cap

**Every request attribute an expression can read is capped at 8 KiB**: each header's value, after its repeats are joined, each query parameter, and each cookie, after perdu decodes its own.
One over the cap is an **oversized attribute**: its name stays, with the value `""` — cutting it arbitrarily at the limit could make its value invalid — and the rest of the request goes on as usual.
8 KiB leaves room for legitimate attributes, hundreds of group names or a large token.
A cookie a browser stores never reaches it, browsers capping a cookie at about 4 KiB, though a crafted one can; a proxy that rejects long header lines, as nginx does over 8 KiB by default, never forwards an oversized header, while proxies built on Go's HTTP server, such as Traefik and Caddy, accept far larger ones.

**A rule computes from that `""` unless it tests `oversized`**, and for `audience` that means a viewer whose group header is too long holds no groups, and sees only the tiles without an `audience`.
To decide otherwise, test the header's name in `oversized.headers`:

```
--audience-expr '"remote-groups" in oversized.headers ? nil : split(headers["remote-groups"], ",")'
--audience-expr '"remote-groups" in oversized.headers ? "guest" : split(headers["remote-groups"], ",")'
```

The first fails open, showing that viewer every tile; the second gives them a default group.
The names are spelled as the maps spell them: lower case for headers.

**Each oversized attribute is logged on every request that runs your rules and carries it**, naming its kind (`headers`, `query`, `cookies`), name, size and the rules that read it: at `WARN` when a rule reads it without testing it, saying that rule is computing from `""` on this request, and at `DEBUG` otherwise, including when no rule reads it — behind a single-sign-on proxy, large headers nobody reads are routine.

> [!TIP]
> These log lines are never collapsed, and you can use their frequency to tell apart one viewer's large header from a client probing; a client can therefore make as many of them as it sends requests.
>
> A guard as shown above, or a narrower rule, moves them to `DEBUG`; meanwhile a log collector can drop them by their message without touching anything else perdu logs.

**Perdu examines each rule once, at startup**, for the names it reads.
A rule that reaches a kind of attribute without naming it (a name computed at run time, or a whole map handed to a function) can't have its reads attributed on a request, so if it doesn't test `oversized` for that kind, startup logs a `WARN` naming the rule and the kinds.
The examination reads the expression's text, so it errs both ways: a name in a branch a request didn't take still counts as read, and a test of `oversized` counts as a guard whatever its branches do — one testing a name written out guards that name, and one testing a name computed at run time its whole kind.

## Ways a rule can surprise its author

* **An oversized attribute reads as `""`** ([The input cap](#the-input-cap)).
* **A name its map can never hold** is refused at startup when the expression writes it out: a header name with capitals, as in `headers["Remote-Groups"]`, a header perdu leaves out, such as `headers["authorization"]`, or the name a rule's own cookie travels under, as in `cookies["perdu_audience"]` beside `--audience-expr`, which reads as `cookies["audience"]`.
  Computed at run time instead, such a name reads like an attribute that wasn't sent, and nothing reports it.
* **A value over the output limit, or a `nil` result, gives no value** ([What a rule gives](#what-a-rule-gives)).
* **`cookies[...]` holds what the browser sent**: the values of the previous page, not those other rules compute for this one.
* **`query` holds what whoever wrote the link chose**, including another site linking to perdu: a viewer following a link from anywhere runs your rules on its query, and keeps the cookies they set, so a rule reading `query` computes what an outsider asks for, which the page shows as text, never as markup.
* **Every value a rule computes is published** to the page and to whatever shares the host's cookie jar ([Every value a rule computes is published to the page](#every-value-a-rule-computes-is-published-to-the-page)).
* **A rule's name hides the cookie sent under that bare name**: `cookies["lang"]` always reads perdu's language cookie, never a `lang` cookie another application sets, and a rule does the same for its own name.
  Removing the rule uncovers that cookie: an expression still reading `cookies["<name>"]` then reads it instead of perdu's, silently.
* **Work isn't a function of input size alone.** The range operator costs whatever value it is given: `1..int(headers["x"])` does a number's worth of work from a six-byte header.
* **A string can grow without limit**: a `reduce` that doubles its accumulator over such a range grows exponentially.
* **Regular expressions can't backtrack catastrophically**: expr's are Go's, which run in linear time.
* **Division is floating point**, so `1/0` is `+Inf` rather than a failure: a rule dividing by something the request supplies gives a strange value instead of none.
* **Iterating a request attribute doesn't compile**, which is the good case: `map(headers["x"], #)` is refused at startup.
  It bites where a string reaches the expression with no known type, out of `fromJSON()` say: iterating it yields bytes, so `map(s, #)` gives numbers rather than characters, and disagrees with `len()`, which counts characters.
  `split(s, "")` walks a string by character.

## The memory budget

Perdu runs each expression with expr's **memory budget set to 100,000**, a tenth of the library's default, and a rule exceeding it fails, giving no value.
**It is a best-effort limit, not a bound**: the budget doesn't charge everything, so a loop concatenating strings, or a `reduce` doubling its accumulator, is charged almost nothing while it allocates without limit, and **a rule can exhaust perdu's memory well inside the budget**.
The budget catches the careless case, not the determined one.
An ordinary `split`, `map` and `filter` over a header stays far inside it, because the input cap bounds what it walks.

Nothing else bounds an expression: a running expression can't be interrupted, so perdu sets no time limit, which could only abandon the work rather than stop it.

> [!CAUTION]
> You are responsible for writing safe expressions, and perdu makes **no security guarantee for an expression that feeds request data into unbounded work**: don't write one whose cost grows with request data you haven't bounded yourself.

## When a rule is refused

Each rule is compiled once, at startup, and perdu refuses to start, naming the rule, when one:

* doesn't compile — a misspelled map name such as `header` included;
* writes out a name its map can never hold ([Ways a rule can surprise its author](#ways-a-rule-can-surprise-its-author)), in any branch, even one no request takes;
* uses `oversized` other than by testing a name in one of its three sets.

A rule that compiles is never refused for the type of value it returns, even where that type is known at startup: a result of a type the tables above refuse is a failure on each request it happens on, logged then.

## A rule named `lang`

A `--data` rule named `lang` writes the language cookie, `<prefix>_lang`, which normally holds the language a viewer chose with the switcher ([Locales](locales.md#which-page-a-viewer-lands-on)): it lets your identity provider decide a viewer's language.
That cookie only decides where `/` sends a viewer, each page's address serving its own language, so the rule's language reaches a viewer who comes in through `/`, never one who opens a page's address directly, from a bookmark or a link.
Know the tradeoff before choosing it:

* the rule runs on every page, so it overrides the switcher: a viewer who picks *Français* lands on the French page, but the rule rewrites their choice there, and their next visit to `/` lands on the rule's language;
* a rule giving `nil` clears a stored choice rather than leaving it;
* its cookie is a session cookie, so on a session's first visit `/` finds no choice and follows the browser's languages, and the rule's language takes over from the next visit to `/`.

Redirecting a user through `/set-lang/<locale>` once, from your identity provider, stores a lasting choice instead, which the switcher can still change.

## Debugging a rule

* At `DEBUG`, every rule logs its outcome on every page request, `rule evaluated`, with its value's size, or `nil`, or `failed`; a page request that isn't a navigation logs `dashboard page served with no rule run`; and a cookie perdu can't decode, which counts as absent, logs `cookie failed to decode`.
* To see what your rules give for a request without a browser, send perdu the headers your proxy would set, and mark the request as a navigation: `curl -si -H 'Sec-Fetch-Mode: navigate' -H 'Remote-Groups: admins' http://localhost:8080/l/en/ | grep -i '^set-cookie'`.
  Each `Set-Cookie` holds a rule's value, percent-encoded after its environment identity and a `!`, and a value ending in `!!` is no value: readable enough to check, not a format to parse.
* In the browser's console, the page warns when a value it reads carries another [environment identity](#the-environment-identity) than the page's own: rare, but the cause of tiles hidden for no visible reason when it happens, as when the browser shows a page from its cache after the labels changed, drops a cookie for its size ([The output limit](#the-output-limit)), or holds a cookie another application or perdu instance wrote under perdu's names ([Deploying perdu](deploy.md#sharing-a-host-name-with-other-applications)).
