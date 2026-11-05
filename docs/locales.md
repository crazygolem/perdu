# Locales

Perdu renders one page per **locale** — a language, or a regional or script variant of one — and sends each viewer to the one that suits them.

A locale is a BCP 47 language tag: `en`, `fr`, `fr-CH`, `pt-BR`, `zh-Hant`.

## Pages

Each locale's page is served from `/l/<locale>/`, its tag in lower case: `/l/fr-ch/` for `fr-CH`.
A page address written in any other case redirects to the lower-case one, and an address without its trailing slash to the one with it.
An address naming a locale with no page leads to the nearest page there is, or to an error page saying there is none ([A locale with no page](#a-locale-with-no-page)).

Every page holds every service; what differs between them is the language of their text and the order of their tiles, which follows each language's alphabet — on a Swedish page, a name starting with `Ä` comes after the names starting with `Z`, where an English page puts it among the `A`'s.

### Which pages exist

**Perdu generates a page for each locale your labels or its chrome ask for, that `--locales` admits.** [`--locales`](flags.md#--locales) is a list of tags and tokens, each admitting locales:

* `~labels`, the default, admits the locales your labels ask for, **as well as the default locale** (so that a stack whose labels carry no locale still has its page);
* `~chrome` admits the locales perdu has [chrome](#chrome) for;
* `~all` admits both;
* a tag admits itself, exactly: listing `en` admits `en` and not `en-GB`, so a value tagged `en-GB` then won't appear on any page, the `en` page reading `en` values.

A label with a locale, such as `<prefix>.description.wae=Mediaserver`, asks for the Walser page, so long as **its value isn't empty** and **its entity isn't set aside**.

**A listed tag that nothing asks for gets no page**, and each build reports it at `INFO`, naming the locale.
The list must admit the default locale: every token does, and a list of tags alone must name it.
A locale perdu has chrome for that the list leaves out is your choice, and nothing reports it; a label asking for a locale the list doesn't admit is logged at `DEBUG`, and every label is still checked, so a mistyped tag is still warned about.

Either is enough: a page needs chrome or a label with its locale, not both.
**A locale only labels name** gets a page with the values tagged with it, its fallback's values for the rest ([Falling back](#falling-back)), and the default locale's chrome, unless a locale it falls back to has chrome of its own.
That is what a partial translation looks like, rather than a page gone wrong: `<prefix>.description.wae=Mediaserver` asks for exactly that.
Each build logs, at `DEBUG`, each locale it generates, the token of `--locales` it expands from, if any, what asked for it, and whether its chrome is the default locale's — the line to look for when the question is "why is my Walser page in English".

**Every page renders every service and lists every page in its language switcher**, so the work a build does grows as pages × (services + pages), not as the translations you write: one description tagged with 300 locales makes 300 pages, each rendering every service and listing 300 pages, on every build.
Under the default, `~labels`, any container's labels can ask for pages, including the labels set by the image ([Labels](labels.md#labels-an-image-declares)), and enough of them make builds slow and large enough to exhaust perdu's memory: **if you run images whose labels you don't vet, list the locales you want** in `--locales`.

**Empty vs unusable**: a label generates its page when perdu reads the labels, before its value is checked ([Labels](labels.md#when-perdu-checks-a-label)): `<prefix>.url.wae=ssh://box` generates `/l/wae/` (because it is not empty), whose tile then links nowhere (because it is unusable), with a `WARN` naming the label.

### The pages' order

The order of the pages is determined by the order of the tags in `--locales`'s expansion: tokens are expanded from left to right, in turn, to the locales they admit that haven't been admitted already by a tag or a previously expanded token.
A later token that adds nothing expands to nothing: in `~all,~labels`, `~all` has admitted every locale `~labels` would; `~labels,~chrome` with a stack that has labels for every chrome locale, `~labels` admits all locales `~chrome` would.
Tokens are expanded in the code-point order of their tags.

> [!TIP]
> To place your preferred language (`fr`) first and a default language (`en`) last, write `fr,~labels,en`: `~labels` will be expanded to all languages named by your labels except `fr` and `en`, so their first and last position is preserved.

That order is the language switcher's **while it lists the locales inline** ([The language switcher](#the-language-switcher)), and it breaks ties when two pages are equally close to what a viewer asks for ([Which page a viewer lands on](#which-page-a-viewer-lands-on)).

### The default locale

`--default-locale` names the locale every page falls back to: one perdu has chrome for, `de`, `en`, `fr` or `it`.

`/` redirects there when nothing a viewer asks for has a page, every page's value falls back to it before the label without a locale, and a page whose language has no chrome takes the default locale's.

## Translating labels

A service's `name`, `description`, `url` and `icon` labels take a locale after the field, and so do an action's `name`, `url` and `icon`: `<prefix>.name.fr`, `<prefix>.actions.password.name.fr`.

```
perdu.name=Foo
perdu.name.fr=Fou
perdu.description=A foo service
perdu.description.fr=Une folie de service
perdu.url=https://foo.example.com
perdu.url.fr=https://foo.example.com/fr
```

**Tags are read in any case, and in their standard form**: `<prefix>.name.fr-ch` and `<prefix>.name.fr-CH` name one locale, and so do `iw` and `he`.
Giving one field two keys for one locale sets its entity aside with a `WARN`, since perdu can't tell which you meant.
These tags are ignored with a `WARN`, as if the key were absent, and the entity stays:

* a tag that doesn't parse, as in `xx-!!`, a tag with a dot, or one with a subtag no registry knows, as in `fr-Qzzz` — which is ignored rather than read as `fr`;
* a code meaning no language: `und`, `mul`, `mis` and `zxx`;
* a tag with an extension or a private-use part, such as `en-u-ca-buddhist`, which could only duplicate a page.

> [!TIP]
> ### Tag text that isn't a name
>
> When a text value's language differs from the page's, perdu marks it with its language so that browsers and screen readers read it correctly: its pronunciation, its hyphenation, the font its script needs.
>
> A label without a locale doesn't declares a language, so each page reads it in that page's own: an untagged English description is read as French on the French page.
>
> So **where text that isn't a name, such as a description, is in the default locale's language, tag it with that locale rather than leaving it without one**: with `--default-locale en`, write `<prefix>.description.en=Films and series` rather than `<prefix>.description=Films and series`.
> Tagged, it is marked as English on every other page; untagged, it is read in each page's language.
> A name reads the same in any language, so it can stay untagged.

## Falling back

Where a page's locale has nothing of its own, whether a label's value or the chrome, the page falls back to each of its parents in CLDR's language data, in order, then to the default locale; a label's value falls back last to the label without a locale.
The two never meet: a label's value never comes from the chrome, nor the chrome from a label, so a page can show its labels in one language and its chrome in another, as a Walser page with Walser labels shows the default locale's chrome.

| Page         | Its parents               |
| ------------ | ------------------------- |
| `fr-CH`      | `fr`                      |
| `en-GB`      | `en-001`, then `en`       |
| `es-MX`      | `es-419`, then `es`       |
| `zh-Hant-TW` | `zh-Hant`, and no further |
| `sr-Latn`    | none                      |

This isn't simply dropping subtags: the parents never cross from one script to another, so a Traditional Chinese page never shows Simplified Chinese text, and moves on to the default locale instead.
A page never borrows a sibling's text either: a `pt-PT` page never shows `pt-BR` labels, so text shared by every Portuguese page goes under `pt`, which both reach.

**An empty value stops the fallback like any other**, which is how you blank one locale: `<prefix>.description.fr=` gives the French page no description, where leaving the key out gives it the default one.
A page's value is chosen when perdu reads the labels, and checked only when it renders the page ([Labels](labels.md#when-perdu-checks-a-label)): a value that fails its check, such as a `url` perdu can't parse or an icon no library answers, leaves the field empty on that page rather than falling back further.

With `--default-locale fr`, labels tagged `fr` provide the values every page falls back to, with no label without a locale:

```
perdu.name.fr=Fou
perdu.description.fr=Une folie de service
```

## Chrome

**Chrome** is the user interface, i.e. the text perdu writes itself around the services: the page's title, the switcher's labels, the placement headings, the empty page's message, etc.

Perdu ships chrome in German, English, French and Italian, and it can't be changed in this release.
A page for any other locale falls back for it ([Falling back](#falling-back)): an `fr-CH` page has the French chrome, a Walser page the default locale's.

## Right-to-left pages

A label naming a right-to-left language, such as `<prefix>.name.mzn=…`, generates a right-to-left page with no chrome of its own: perdu lays it out mirrored, the switcher and the footer at the left, icons right of the names.

A page's direction comes from its locale's name in its own language, e.g. `العربية` for `ar`.
A locale perdu knows no name for is shown by its tag and laid out left to right, which suits most such locales but not the right-to-left ones, such as `dv` (Dhivehi) or `syr` (Syriac): their text still runs right to left, but their pages align and punctuate as left-to-right ones, and this release has no way to set a page's direction.

## The language switcher

With two locales or more, the page offers a link to each, listed inline up to five locales and as a dropdown from six ([The template](template.md#the-language-switcher)).

**The switcher's order**: listed inline, the locales keep the pages' order as set in `--locales`' ([The pages' order](#the-pages-order)); in the six-or-more dropdown, they are in the code-point order of their tags, the same on every page.

The switcher's links go to `/set-lang/<locale>`, which stores the viewer's choice in a cookie, `<prefix>_lang`, for 365 days, and redirects to that locale's page.

## Which page a viewer lands on

`/` redirects each viewer to a page, choosing, in order:

1. **the viewer's stored choice**, if it still matches a page: a viewer who chose `fr-CH` lands on `fr` once the `fr-CH` page is gone, rather than on what their browser asks for;
2. **the languages their browser asks for**, `Accept-Language`, in the order of preference it gives, each matched to the closest page there is: the same locale, a less specific one, or a close relative — a viewer asking for `pt` lands on `pt-BR` when no `pt` page exists, and one asking for `pt-AO` on `pt-PT`.
   A match never crosses scripts: a viewer asking for `zh-Hant` never lands on a Simplified Chinese page, and moves on to their next language;
3. **the default locale**, when nothing above matches.

Two pages equally close to what a viewer asks for are decided by the pages' order, `--locales`' ([The pages' order](#the-pages-order)), the earlier winning: with no `de` page, a viewer asking for `de` lands on `de-CH` where `--locales` lists it before `de-AT`, and on `de-AT` under the default, `~labels`, which places it first by code point.

A viewer opening `/l/fr/` directly — a bookmark, a shared link — sees the French page without changing the locale they chose: only `/set-lang/` stores a choice.
**Any navigation to `/set-lang/<locale>` stores it, including one that starts on another site**.
The worst a hostile link can do is show a viewer another of your pages, which the switcher undoes.

> [!TIP]
> You can link a new user to `/set-lang/fr` instead of `/l/fr/` to set their language, for example in an onboarding email, and an identity provider that knows a user's language can redirect them through it.

`/set-lang/` accepts any spelling of a locale, `/set-lang/FR`, or `/set-lang/iw` for `he`, and **stores any locale that can have a page, whether or not one exists**: a viewer following `/set-lang/fr-CH` while only a `fr` page exists stores `fr-CH` and reads the `fr` page, through `/l/fr-ch/` ([A locale with no page](#a-locale-with-no-page)), then lands on the `fr-CH` page from `/` once one exists.
A value no page can have — a malformed tag, one naming no language such as `und`, or one carrying an extension — stores nothing, and gets an error page saying there is no page in the default locale's language.

A cookie is only ever set by a navigation: an icon pointing at `/set-lang/fr` sets nothing, and a browser prefetching the link sets nothing until the viewer actually follows it.
To drive the language from your identity provider on every page instead, see [A rule named `lang`](rules.md#a-rule-named-lang).

## A locale with no page

Locales may come and go, as you update tagged labels and bring services up and down.
A viewer refreshing a page, or requesting a page from its direct link — from a bookmark, written by hand — could find that the page is no longer available.

For such cases, perdu looks for the nearest page:

* **The closest page there is**, by a redirect: the address's locale is matched to the pages as a viewer's stored choice is at `/` ([Which page a viewer lands on](#which-page-a-viewer-lands-on)), so it reaches a less specific locale or a close relative.
  `/l/fr-ch/` leads to `/l/fr/` while no `fr-CH` page exists, and to its own page once one does; `/l/pt/` to a `pt-BR` page while no `pt` one exists; an alias to its language's page, `/l/iw/` to `/l/he/`.
  An address naming no language is matched too: `/l/und-ch/` leads to a `de-CH` page where one exists.
  The default locale's page is reached only where it matches, never as a fallback, so a locale with no relative among the pages isn't shown another language silently.
* Otherwise, a **[missing-locale error page](template.md#the-missing-locale-error-page)** saying there is no page in the requested language, answered with a `404`.
  It is in the language of the locale or of its first parent perdu has [chrome](#chrome) for, and in the default locale's otherwise: `/l/de-ch/` reads in German where no German page exists, `/l/wae/` in the default locale's language, and so does an address naming no language that matches no page.

> [!NOTE]
> Each build writes that missing-locale error page for every locale perdu has chrome for, whether or not it has a page.

The missing-locale error page is shown when perdu **could have otherwise answered with a page**: an address with an invalid tag character, such as `/l/en_GB/`, gets a plain `404`, as any other unknown path does.
