# Flags and environment variables

Perdu is configured by flags on its command line, each of which but `--data` can be set by an environment variable instead.

`perdu -h` gives a summary of this document.

## Giving flags

Write a flag with two dashes, its value after `=` or as the next argument: `--icons-dir=/icons` or `--icons-dir /icons`.

`-h` or `--help` prints the usage text and exits with status 0, starting nothing.
It is read in its place among the arguments, so anything the parse refuses before it — an unknown flag, a repeated one, a stray argument — is reported instead; a flag's value is checked only once every argument has been read, so a value perdu would refuse doesn't stop it.

**Perdu refuses what it can't read unambiguously, and stops at the first refusal**, logging an `ERROR` that names the flag, the value or the entry at fault, and where the value came from:

* an unknown flag, or any argument that isn't a flag;
* a flag given more than once, even with the same value — `--data` apart, which is given once per rule;
* a value the flag's own check refuses, which each flag's entry below describes.

The startup sequence logs every flag's value and where it came from at the `INFO` level: `flag`, `env` or `default` ([Reading the logs](logs.md#startup)).

A value is used exactly as written, and is never trimmed as a whole: a value of only spaces is not an empty value, and fails the flag's check.
A **list** flag takes its entries comma-separated, each trimmed of spaces around it, so `en, fr` is `en` and `fr`; an empty entry, as in `en,,fr` or a trailing comma, is refused, naming the entry.
What an **empty value** means is each flag's own, and its entry says: unset for some, the empty list for others, refused for the rest.

> [!TIP]
> A value may need quoting or escaping to reach perdu as you wrote it: a shell, a compose file or an `env_file` each reads it first, by its own rules.
> Expressions are the values most likely to need it, as [Deploying perdu](deploy.md#compose-stack-example) shows for a compose file.

## Environment variables

Every flag but `--data`, which is given once per rule, can be set by an environment variable: `PERDU_` then the flag's name in capitals, with `_` for `-`, such as `PERDU_ICONS_DIR` for `--icons-dir`.

The variable is a layer beneath the command line: **a flag given on the command line wins over its variable**, whatever either says.
That lets a variable an image or a shared `env_file` sets be overridden for one container by its command line.

A variable **set but empty** gives the flag its empty value, just as the flag given empty would, rather than leaving the flag unset: `PERDU_IDENTITY_LABELS=` is the empty list of identity labels.
Conversely, a flag given empty on the command line undoes its variable where the empty value means unset: `--icons-dir=` turns off an icons directory `PERDU_ICONS_DIR` names.

A `PERDU_` variable no flag reads — `PERDU_DATA` among them — is ignored, and one `WARN` at startup names every such variable, so that a mistyped name doesn't go unnoticed.
The prefix is always `PERDU_`, whatever `--namespace` says.

Where perdu refuses a combination of two flags, it refuses it whichever of the two came from the command line and whichever from a variable.

Docker's own `DOCKER_*` variables set the connection to the docker API ([Deploying perdu](deploy.md#access-to-the-docker-api)), and `HTTPS_PROXY`, `HTTP_PROXY` and `NO_PROXY` the proxy `di:` icons are fetched through ([Deploying perdu](deploy.md#fetching-icons-through-a-proxy)).
Those variables don't have a corresponding flag; they can only be set through the environment.

## Reference

### What perdu reads from docker

#### `--namespace`

The prefix of every label key perdu reads and every cookie it sets: `perdu.name` and `perdu_lang` under the default.
Two perdu instances watching one daemon with different namespaces each read only their own labels, and two on one host need different namespaces so that their cookies don't collide ([Deploying perdu](deploy.md#sharing-a-host-name-with-other-applications)).

* Variable: `PERDU_NAMESPACE`
* Default: `perdu`
* Value: `[a-zA-Z0-9-]+`.
  Anything else is refused, the empty value included: a dot would split a label key and an underscore a cookie name, and a browser silently drops a cookie whose name it can't read.

#### `--identity-labels`

The label keys whose values decide which containers make one service: containers carrying the same value for every key listed are one service, however many there are.
The default groups the containers of one compose service, its replicas included, into one service.
[Labels](labels.md#service-identity) explains what follows from it.

* Variable: `PERDU_IDENTITY_LABELS`
* Default: `com.docker.compose.project,com.docker.compose.service`
* Value: a list of label keys.
  A container missing any key listed is never on the page, and perdu logs it at `INFO`.
* Empty value: the empty list, which makes each container a service of its own, identified by its container ID.
  A compose service scaled to three replicas is then three identical tiles, which move relative to each other whenever a container is recreated.
* Refused: a key listed twice; a key containing `=`, which docker's label filter can't express; a key under the namespace, such as `perdu.group`.
  A key containing a comma can't be listed: it arrives as two keys, which no container carries.

### The pages

#### `--default-locale`

The locale every page falls back to: the language of `/`'s redirect when nothing the viewer accepts has a page, the last step of every translation's fallback before the label without a locale, and the source of the [chrome](locales.md#chrome) a page's own language lacks ([Locales](locales.md)).

* Variable: `PERDU_DEFAULT_LOCALE`
* Default: none; perdu doesn't start without it.
* Value: a language tag perdu has chrome for — `de`, `en`, `fr` or `it` — in any case.
* Refused: a tag `--locales` refuses; any other locale, even one your labels name.

#### `--locales`

The locale pages to generate, as a list of locales and of tokens standing for the locales your labels or perdu's [chrome](locales.md#chrome) ask for ([Locales](locales.md#which-pages-exist)).
By default, perdu generates a page for the default locale and each locale your labels ask for, rather than one for every language it has chrome for, since every page renders every service; `~chrome`, `~all` and tags ask for more, and a list of tags alone bounds the pages to the locales it names.
Under the default, an image's own labels ask for pages too, and a build's work grows with the square of the pages: if you run images whose labels you don't vet, list the locales you want.

* Variable: `PERDU_LOCALES`
* Default: `~labels`.
* Value: a list of language tags, in any case, and tokens, which admit locales:
  * a tag admits itself, exactly: listing `fr` admits `fr`, not `fr-CH`;
  * `~labels` admits each locale a label of an included entity names with a value that isn't empty, and the default locale;
  * `~chrome` admits each locale perdu has chrome for;
  * `~all` admits both.

  Perdu generates a page for each locale a label or its chrome asks for that the list admits; a listed tag nothing asks for generates no page, which perdu reports at `INFO` on every build.
  The list must admit the locale `--default-locale` names: every token does, and a list of tags alone must name it.
  The pages are in the list's order, which the language switcher keeps while it shows them inline and which decides between two pages equally close to what a viewer accepts ([Locales](locales.md#the-pages-order)).
* Empty value: refused, since the default has a spelling of its own, `~labels`.
* Refused: a tag that doesn't parse, or names no language, such as `und`; a tag with an extension, such as `en-u-ca-buddhist`; an entry starting with `~` that isn't one of the tokens, spelled as above, as in `~Labels` or `~label`; a locale listed twice, in any spelling, as in `fr-ch,fr-CH` or `iw,he`, or a token given twice, though a tag a token also admits is no repeat; and a list that doesn't admit `--default-locale`.

> [!IMPORTANT]
> In a shell, quote a value that is a token alone, as in `--locales '~chrome'` or `PERDU_LOCALES='~all'`.
> A shell reads a word starting with `~` as a home directory: unquoted, `~chrome` becomes the home directory of a user named `chrome` where there is one, and where there is none, a shell such as zsh refuses the command.
> A token after a comma, as in `en,~chrome`, reaches perdu as written.

#### `--icons-dir`

A directory holding icon libraries of your own, one subdirectory per library, which also overrides the built-in ones ([Icons](icons.md#your-own-icons)).

* Variable: `PERDU_ICONS_DIR`
* Default: none, and no icon is read from a directory.
* Value: the directory's path inside perdu's container, where you mount it.
  At startup it must exist and be a readable directory within the startup window, or perdu doesn't start; an empty one only logs a `WARN` ([Deploying perdu](deploy.md#starting)).
* Empty value: unset.

### Serving

#### `--listen`

The addresses perdu listens on.

* Variable: `PERDU_LISTEN`
* Default: `:8080`
* Value: a list of addresses, each a host and a port joined by a colon: `:8080`, `127.0.0.1:8080`, `[::1]:8080`, `perdu.internal:8080`.
  An empty host listens on every address of both IPv4 and IPv6, `0.0.0.0` on every IPv4 address only, and `[::]` on every IPv6 address only.
  A host name is resolved at startup, and perdu listens on every address it resolves to.
  An address given twice, in any spelling or through a name, is listened on once.
* Refused: the empty value; an entry that isn't a host and a port; a port that isn't one.
  An address the system refuses to listen on, a name that doesn't resolve, and a lookup taking more than 5 seconds each stop perdu at startup.

#### `--base-path`

The path perdu is served under, when it shares its host name with other applications ([Deploying perdu](deploy.md#paths)): under `--base-path=/dashboard`, the pages are at `/dashboard/l/en/` and so on, and perdu's cookies are scoped to `/dashboard`.
Your proxy forwards the full path, base path included.

* Variable: `PERDU_BASE_PATH`
* Default: `/`, the root.
* Value: a path, its leading and trailing slashes optional: `dashboard`, `/dashboard`, `dashboard/` and `/dashboard/` are one base path, and `/` is the root.
  Each segment must match `[a-zA-Z0-9._~-]+`.
* Refused: the empty value (the root is written `/`); a segment that is `.` or `..`, or holds any other character; an empty segment, as in `/a//b`, `//dashboard` or `dashboard//`.

#### `--trusted-proxies`

The proxies whose `X-Forwarded-For` perdu believes when it logs a request's client address.
Perdu reads the client address for nothing but its log lines.
Every address an entry covers can choose the address logged for its own requests: list your proxy's own rather than its network's range ([Deploying perdu](deploy.md#the-clients-address)).

* Variable: `PERDU_TRUSTED_PROXIES`
* Default: none: every line names the address that connected to perdu.
* Value: a list whose entries are each an address (`10.0.0.2`), a CIDR range (`10.0.0.0/24`, `fd00::/8`), or a range of two addresses of one family joined by `-` (`10.0.0.2-10.0.0.9`).
  Entries may repeat or overlap.
  An IPv4-mapped IPv6 address (an IPv4 address written as an IPv6 one) is the IPv4 address, and an IPv6 range written that way, `::ffff:10.0.0.0/104`, the IPv4 range it covers; any other IPv6 range holds IPv6 addresses only, so even `::/0` trusts no IPv4 proxy.
* Empty value: the empty list.
* Refused: an entry that is in none of the three forms; an entry that carries a zone, such as `fe80::1%eth0` (a zone names a link only from the side of whoever wrote it); a range whose first address is above its last, or whose ends are of different families; a range written with IPv4 addresses in IPv6 form shorter than `/96`, which would span both families.

#### `--cookie-secure`

Whether perdu's cookies carry the `Secure` attribute, which keeps a browser from sending them over plain HTTP.

* Variable: `PERDU_COOKIE_SECURE`
* Default: `auto`
* Value: `auto`, which marks a response's cookies `Secure` when its request's `X-Forwarded-Proto` says the viewer reached your proxy over HTTPS ([Deploying perdu](deploy.md#the-clients-scheme)); `true`, which always marks them; or `false`, which never does, and is almost always a mistake in production.

### Per-viewer data

#### `--data`

A value perdu computes on each page request from an expression, and hands to the page's scripts as a cookie.
The built-in template uses one, `name`, to greet the viewer ([Per-viewer data](rules.md#the-greeting)).

* Variable: none.
* Default: none.
* Value: `<name>=<expr>`, one argument split at its first `=` (the expression may hold `=` itself): `--data 'name=headers["remote-name"]'`.
  Give the flag once per name.
  The name must match `[a-zA-Z0-9_-]+`; it names the cookie, `<prefix>_<name>`.
* Refused: an argument without `=`, an empty name, or one outside that set; a name given twice; an expression that doesn't compile, or names a request attribute that can never be there ([Per-viewer data](rules.md#when-a-rule-is-refused)); a rule named `audience` if `--audience-expr` is set.

#### `--audience-expr`

The expression computing the viewer's groups, which the page matches against each tile's `audience` label to hide the tiles not meant for them ([Per-viewer data](rules.md#the-viewers-groups)).
It works as a `--data` rule named `audience` whose result perdu additionally checks and cleans for the page.

* Variable: `PERDU_AUDIENCE_EXPR`
* Default: none; without it and without a `--data` rule named `audience`, filtering is off, and every viewer sees every tile ([Per-viewer data](rules.md#the-viewers-groups)).
* Value: an expression giving a string of comma-separated groups, a list of strings, or `nil`.
* Empty value: unset.
* Refused: an expression that doesn't compile, or names a request attribute that can never be there; any expression, if a `--data` rule named `audience` is given.

### Logging

The two log flags are read first, so that every later startup error is logged as they say; an error in either is logged in the default level and format ([Reading the logs](logs.md)).

#### `--log-level`

The lowest level logged.

* Variable: `PERDU_LOG_LEVEL`
* Default: `info`
* Value: `debug`, `info`, `warn` or `error`, in lower case.

#### `--log-format`

The log's format, written to standard error.

* Variable: `PERDU_LOG_FORMAT`
* Default: `text`
* Value: `text`, lines of `key=value` pairs, or `json`, one object per line.
