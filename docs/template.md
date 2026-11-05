# The template

Perdu renders its pages with a built-in template.
In this release, the template can't be replaced and its look can't be changed.

The page's styles sit in three cascade layers of perdu's, declared before any other stylesheet: `perdu-held`, the few rules the page's features rest on, such as the one hiding a tile from a viewer, the bolt of an action without an icon and the spacing of a plate's letters; then `perdu-defaults`, what each part needs to read as itself where no stylesheet styles it; then `perdu-theme`, the built-in stylesheet.
So a stylesheet a page gains — a browser extension's, for instance — overrides the defaults and the theme with any of its rules, whatever their specificity, in a layer of its own or in none, while the rules of `perdu-held` hold against it, `!important` or not: only a viewer's own `!important` rules override those.

This chapter describes the template's features, and how they behave for a viewer.

## Sections

The template groups tiles into sections, following each entity's [`category`](labels.md#category) label.
The uncategorized section (`~default` placement) comes first, then the sections your categories name, then the administration section (`~admin` placement).

Only the administration section has a heading; categories are currently not translatable, so their sections have no heading.

Within a section, services come first, then the actions standing on their own ([Actions](#actions)), each in the page's collation order.

**A section with no tile visible to the viewer is hidden.**
When no tile is visible at all, because nothing was discovered or everything is hidden from this viewer, the page shows a fallback message which assistive technology reads as "No services to show".

## Tiles

### Links

A tile links to its entity's `url`, which opens in the same tab.

A tile without a usable [`url`](labels.md#url) links nowhere, and is drawn dimmed so that viewers can tell it apart from the tiles they can follow.

### The second line

Under its name, a tile shows its description, or its URL when it has no description.
A tile with no description and no URL to show has no second line.

URLs are shown with only their host and path, decoded for reading: a host in Punycode is shown in Unicode, and a percent-encoded path is shown decoded.
For root-relative URLs, perdu prepends the host from which the viewer reached perdu.

### Long text

Descriptions longer than 5 lines and URLs that don't fit their line are ellipsized.

Their full content is shown in a tooltip on hover, and expanded while their tile has keyboard focus.
On a tile without a link, the description is also expanded when clicked or tapped, and reached with Tab: a screen reader then introduces it as a description and reads it whole, the cut being only visual.
On a touch screen, tapping a linked tile follows its link, so its description can't be shown in full.

Service and action names are never ellipsized.

### Actions

A service's actions are listed in its tile, in the order of their keys ([Labels](labels.md#actions)).

An action placed in a section its service isn't in gets a tile of its own, with its service's name as its second line; screen readers introduce that name with "Provided by:".

### Icons

Services with a missing icon get a plate holding the service's initials, in a colour and a shape kept for as long as the labels identifying the service stay the same ([Icons](icons.md#how-the-page-draws-icons)).
An action without an icon shows a bolt.

Monochrome icons are tinted in the page's text colour.

See [Icons](icons.md#how-the-page-draws-icons) for a detailed description of perdu's icon handling.

## The language switcher

With two locales or more, a language switcher is shown to let the viewer navigate between the page's languages; with one, there is no switcher.

* **Two to five locales are listed inline**, by their names in their own languages, such as "Deutsch", "English" and "Français", or by their tags where the names don't fit.
  A tag shows its locale's name in a tooltip, on keyboard focus and on hover where the pointer can hover: not on a touch screen, where tapping a tag switches language at once.
  The order is `--locales`', so you choose it.
* **Six or more generate a dropdown**: a button showing the current language, which opens a list of every locale by name.
  The order is the code-point order of their tags, the same on every page ([Locales](locales.md#the-language-switcher)).

Perdu infers the language's name from its tag; a tag perdu can't map to a name is listed by its tag.

See [Locales](locales.md#the-language-switcher) for the exact sort order, and for how a viewer's choice of language is remembered.

## Per-viewer features

The template uses the values your rules compute for each viewer ([Per-viewer data](rules.md)) for two features.

**The greeting**: the page greets the viewer by name, e.g. "Welcome, Ada!", from the value named `name` ([Per-viewer data](rules.md#the-greeting)).
Without that value, there is no greeting.

**Filtering**: with `--audience-expr`, the page hides each tile whose `audience` names none of the viewer's groups, before showing the page ([Per-viewer data](rules.md#the-viewers-groups)).
An action listed in a hidden tile is hidden with it, and a section left with no visible tile is hidden too.

> [!IMPORTANT]
> Every page still holds every tile: filtering is presentational, not access control.

## Colour schemes

The template has a light and a dark colour scheme, and **follows the viewer's system or browser setting** (`prefers-color-scheme`): there is no switch on the page itself.

To see the dark scheme, set your system's appearance to dark, or your browser's appearance for websites, where it has such a setting.
The developer tools of Firefox and Chromium-based browsers can also emulate `prefers-color-scheme: dark` for one tab.

## Other system preferences

The template also follows these settings:

* **Forced colours**, as set by a Windows contrast theme or a browser overriding page colours: the page takes the system's colours, and links its link colour.
* **More contrast** (`prefers-contrast: more`): dimmed text, lines and the accent colour take the full text colour, and so do the initials' plates.
* **Reduced motion**: nothing on the page moves, and hover and focus take effect at once.
* **Text size**: every size follows the browser's default text size.

The template uses the system's own fonts, so that every language's script shows in a face made for it.

## Accessibility

The template targets WCAG 2.2, level AA, which includes:

* text contrast;
* a visible focus ring on every link, in reading order;
* link targets large enough to hit;
* no sideways scrolling at 200% zoom or on a 320-pixel-wide screen;
* landmarks for the header, the switcher, the main area, the administration section and the footer;
* each part's language marked where it differs from the page's.

Two gaps are accepted:

* **Text without a locale is read in each page's language**, not in the default locale's; explicitly tag text that isn't a name with the default locale ([Locales](locales.md#tag-text-that-isnt-a-name)).
* **Sections have no name**, the administration section aside, since categories are currently not translatable.
  Each section's list tells assistive technology which tiles belong together.

## Graceful degradation

Perdu targets up-to-date browsers that support modern CSS and JavaScript features.
The built-in template degrades gracefully when features it uses are not yet supported by a browser; the page may lose cosmetic features, but should not look wrong.

The page stays usable without JavaScript, and under a `Content-Security-Policy` that refuses part of what it needs: links and the language switcher keep working, and the features depending on what's missing are lost.

### Without JavaScript

Without JavaScript:

* every viewer sees every tile, since nothing hides them;
* there is no greeting;
* the inline switcher lists the locales by their tags, with no tooltips, and the dropdown's button is only as wide as the current language's name, so the header may shift from one language's page to the next;
* every icon shows in its own colours;
* a root-relative URL shows its path alone;
* a tile without a link shows its description whole;
* cut text has no tooltip.

### Under a Content Security Policy

A `Content-Security-Policy` your proxy adds removes the parts of the page that depend on what it refuses ([Deploying perdu](deploy.md#content-security-policy)):

* **Refusing inline scripts** gives the page [without JavaScript](#without-javascript).
* **Refusing inline styles** makes filtering fail open and changes how parts of the page are drawn: every viewer sees every tile, the empty page's fallback message shows on a page that has tiles, the bolt is missing, and every icon shows in its own colours.
  Also, a failed icon shows what the browser draws for a broken image, the language dropdown opens in its default position, a plate's letters follow a viewer's text-spacing override, and in Firefox, under forced colours, links show in the text colour.
* **Refusing `data:` images** removes the bolt.

## The missing-locale error page

The missing-locale error page is shown when a page for a valid locale is requested, but it [hasn't been generated](locales.md#a-locale-with-no-page) — no corresponding chrome nor tagged labels, or filtered by `--locales` — and perdu doesn't have an appropriate fallback page.

It keeps the layout and styling of the normal page — the header with the greeting and language switcher, the footer — and shows in place of the sections a line stating the issue and a link to `/`, which leads the viewer to their page following the standard language resolution process.

Unless the viewer uses the language switcher, the language cookie is not updated and the viewer can land on the normal page again on the next visit if the page has been generated meanwhile.

> [!NOTE]
> **Rules are not run for error responses**, including the missing-locale error page: the greeting uses the viewer's `name` from a previously served page, if any.
