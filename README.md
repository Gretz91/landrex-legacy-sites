# Landrex legacy sites

Near-pixel-faithful archival copies of Landrex's old WordPress/Elementor microsites, built as plain static HTML/CSS. This is a **separate, parallel effort** from the main [landrex-web](https://github.com/Gretz91/landrex-web) consolidated site — not a redesign, not folded into that build.

## Why this exists

- The new landrex-web site's outbound links will point here instead of at the real external domains (e.g. tanglewoodridge.com).
- The old domains are meant to eventually CNAME/redirect here too.
- Each copy keeps working forms (routed to landrex-web's `/api/lead`, CORS-allowlisted) and working image galleries/carousels, but drops Facebook Pixel/GTM tracking and the live WordPress/Elementor/Gravity Forms backend these pages used to depend on.

## Structure

Each site is a plain static folder at the repo root, deployable as-is (no build step):

```
/<site-slug>/
  index.html
  style.css
  assets/
    images/
    fonts/
    docs/        (only if the site links to a PDF or similar, self-hosted alongside everything else)
  vendor/        (only on sites that depend on a JS library like jQuery/Slick, self-hosted rather than pulled from a CDN)
```

`index.html` references `style.css` and `assets/` with paths relative to its own folder, so each site is self-contained and can be opened directly or served from any static host.

## Sites

| Slug | Original domain | Status |
|---|---|---|
| tanglewood-ridge | tanglewoodridge.com | ✅ done, verified against live site |
| erin-ridge-north | erinridgenorth.ca | ✅ done, verified against live site |
| ridgeview-at-the-quarry | ridgeviewatthequarry.ca | ✅ done, verified against live site |
| windsor-pointe | windsorpointe.ca | ✅ done, verified against live site |
| sherwood-golf-estates | sherwoodestates.ca | ✅ done, verified against live site |
| east-pine | eastpine.ca | ✅ done, verified against live site |
| heritage-valley-estates | heritagevalleyestates.ca | ✅ done, verified against live site |
| westwinds | westwindsliving.ca | ✅ done, verified against live site |
| heartland-meadows | theheartlandmeadows.com | ✅ done, verified against live site |
| fairways-north | fairwaysnorth.com | ✅ done, verified against live site |
| east-village | east-village.ca | ✅ done, verified against live site |
| urban-district | urbandistrict.ca | pending |
| mirigal | mirigal.ca | pending |

Excluded: `livefenwyck.com` (owned by Avillia Developments, not Landrex), `mail.theheartlandmeadows.com` (email only), `forestheightsbeaumont.landrex.com` (unreachable/broken, not a live site).

## Build method

Documented in full in the Landrex project's `claude/legacy-sites-status.md` doc. Short version per site:

1. Capture the live page's DOM + only the CSS rules it actually uses ("critical CSS extraction"), self-host its images/fonts (and any linked PDFs/docs), and package as a bundle.
2. Strip every script tied to the old CMS and its trackers: Facebook Pixel, GTM/GA, Microsoft Clarity, Elementor/Beaver Builder runtime, Gravity Forms, Popup Maker backend calls. Remove the now-dead `<link>` tags pointing at the live wp-content assets.
3. Re-add the `<link rel="stylesheet" href="style.css">` the bundler doesn't insert automatically.
4. Rewrite the contact form(s) to post JSON to `https://landrex-web.vercel.app/api/lead` with a honeypot field, instead of the old Gravity Forms action.
5. Restore any working gallery/carousel with minimal vanilla JS. If the page genuinely depends on a JS library (e.g. Slick Carousel + jQuery), self-host that library under `vendor/` instead of pointing at a CDN — more robust for an archival copy, and the bundler snapshot often already has the plugin's generated markup (cloned slides, inline positioning) baked into the DOM, which has to be unwound back to the library's pre-init markup before the page's own init script can run cleanly against it a second time. A true third-party service dependency with no tracking/CMS backend (Google Maps embed, ScrollMagic, Font Awesome's kit loader) is fine to leave as a live CDN/API reference.
6. Verify: no tracking/CMS leftovers, HTML parses cleanly, zero browser console/page errors, carousels actually paginate, screenshot comparison against the live site, test-submit the form with the honeypot filled.

A CSS file with a *relative* `url(...)` reference (`../img/x.jpg`, `./fonts/y.woff`) needs that path resolved against the stylesheet's own URL, not the page's — and the asset's self-hosted path needs to be written back into the shipped CSS as a full absolute-URL match, not left as the original relative text, or it silently 404s in the browser even though the bundler fetched the right bytes. Caught and fixed in the bundler itself (`scripts/bundler.js`) on Windsor Pointe, where it broke Font Awesome's icon fonts and two background images.

Two more generalizable lessons surfaced on Sherwood Golf Estates:

- **Lazy-loaded images can be invisible to the bundler, not just mis-resolved.** Some WordPress lazy-load setups put the real URL in `data-lazy-src`/`data-src` (`<img>`) and `data-lazy-srcset`/`data-srcset` (`<source>`), leaving `src`/`srcset` pointing at a blank placeholder. The bundler's `img[src]` scan never sees the real URL at all. Fix: before calling `__bundleSite`, run a DOM pre-pass that copies `data-lazy-src`/`data-src` into `img.src` and `data-lazy-srcset`/`data-srcset` into `srcset`, for every element that has them — after which the bundler's existing collection logic picks up the real values with no bundler.js changes needed. This has not yet been folded into `bundler.js` as a standing step, so **remember to run it manually on every future site** (any WP site with a lazy-load plugin can hit this).
- **`rewriteAll()`'s substring replace can "localize" a path that was never fetched.** The bundler never scans `<picture><source srcset>` at all, only `<img src>`. When a registered URL (the `<img>` fallback's `.jpg`/`.png`, which *was* fetched) is a literal text prefix of an unregistered sibling URL elsewhere in the page (a `.jpg.webp`/`.png.webp` variant in a `<source>`, which was *not* fetched), the plain `.split(v).join(local)` substitution still matches inside that longer string — producing a plausible-looking local path (`assets/images/x.jpg.webp`) for a file that doesn't exist on disk. Only caught via Playwright's `requestfailed` listener. Fixed per-site by stripping the orphaned `<source type="image/webp">` tags (the `<img>` fallback already renders correctly); **`bundler.js` itself still needs a permanent fix** — either collect+fetch `srcset` candidates, or make `rewriteAll` collision-safe against prefix matches — before the next `<picture>`/webp site.

East Pine (a different stack entirely — plain Elementor/Hello Elementor, no Gravity Forms, no Slick/Swiper) surfaced one more variant of the same family of bugs:

- **Elementor's own gallery widget lazy-loads background images via a `data-thumbnail` attribute + an IntersectionObserver, not `data-lazy-src`.** The bundler's `img[src]`/`[style*=background]` scans only see what's already in the DOM at snapshot time; a `.e-gallery-image` div with `class="...e-gallery--lazyload"` has no `style` attribute at all until it scrolls into view. Fix: before bundling, force-set the inline style directly from the known attribute (`el.style.backgroundImage = \`url("${el.dataset.thumbnail}")\``) rather than relying on scrolling to trigger the observer (scrolling worked for only 11/12 items in testing — one straggler never fired). This is the same *shape* of bug as Sherwood's `data-lazy-src` (a real URL sitting in a non-standard attribute, invisible to the bundler's fixed scan list) but a different attribute name and a different triggering mechanism (CSS lazy-load via observer, not swapped `src`) — confirms this is a recurring category (**check any lazy-loading site for where the real URL actually lives**, not just the two known attribute names) rather than a one-off.
- One gallery image (`DSCF8420`) was already 404ing on the **live site itself** (confirmed via direct fetch to both its thumbnail and full-size URLs returning 404 pre-bundling) — not a bundler or archival-copy bug, just a dead upload on the original WordPress site. Left as-is; the archival copy faithfully reproduces the same broken thumbnail the live site shows.

Heritage Valley Estates (Happy Addons + native Elementor widgets, newer Gravity Forms markup) surfaced a third carousel-widget combination and the first "gravity-theme" form markup:

- **Three distinct carousel/gallery mechanisms on one page.** (1) Happy Addons' `ha-slider` is a genuine Slick carousel (`hajs-slick ha-slick` classes) with no surviving init script once the Happy Addons runtime is stripped as CMS backend — rebuilt with a hand-written Slick init (self-hosted vendor jQuery+Slick, same approach as Sherwood's map-slider), reusing the page's own arrow buttons via Slick's `prevArrow`/`nextArrow` options. **Important Slick gotcha**: the container must hold the raw slide elements directly — Slick builds its own `.slick-list`/`.slick-track` wrapper from plain children at init time. An initial attempt that pre-built that wrapper structure by hand (reasoning it would mirror the captured snapshot) silently broke Slick's internal slide-count math: both arrows ended up permanently `slick-hidden`/`aria-disabled` after init. Fixed by leaving the slides as flat children and letting Slick build its own wrapper. (2) Elementor's native Swiper "Slides" widget, same duplicate-filtering problem as Ridgeview at the Quarry — 3 real slides identified by their unique `elementor-repeater-item-<hash>` class (not by `swiper-slide-active/-prev/-next`, which repeat across each loop-clone set), rebuilt as the same small vanilla-JS fade carousel used on Ridgeview. (3) Elementor's native masonry gallery widget, same `data-thumbnail` lazy-load pattern as East Pine's gallery — already fixed by the pre-bundling pre-pass, needing no further JS since this particular gallery has no lightbox attributes at all, just a static photo grid.
- **A hidden, inert reCAPTCHA "bubble" iframe shell baked into the snapshot outside the form itself.** Google's reCAPTCHA v2 widget injects a separate floating challenge-popup container directly into `<body>` (`visibility:hidden; opacity:0`, never actually shown), independent of the inline widget markup inside the form. It's invisible but its iframe `src` still fires a live `api2/bframe` network request on every page load. Caught via Playwright's `requestfailed` listener; stripped outright since the rewritten form drops the CAPTCHA field entirely.
- **First sighting of Gravity Forms' newer "gravity-theme" markup** (`gform-theme--no-framework`, `<div class="gfield">`/`<fieldset>` instead of the `<li id="field_X_Y">` structure seen on every prior Gravity-Forms site). Rewritten the same way as before — POST JSON to `/api/lead` — combining the split First/Last name fields into one `name` value and dropping both the reCAPTCHA field and the Akismet honeypot markup (`ak_hp_textarea`/`ak_js`), neither of which the rewritten form needs.

Westwinds (same "easton"/Foundation-theme family as Erin Ridge North/Windsor Pointe/Sherwood, just a different per-site theme slug) turned out to be the simplest build yet, but with one genuine misdirection worth noting:

- **A theme-wide carousel init script can be present in the page's `<script>` tags with no matching elements anywhere on that particular page.** This homepage's surviving (non-CMS-runtime) inline script called `$('.map-slider').slick(...)` and `$('.builder-slider-left').slick(...)` — the exact same synced-pair pattern as Erin Ridge North/Windsor Pointe/Sherwood — which looked at first glance like it needed the same vendor jQuery+Slick self-hosting treatment. But a live DOM query (`document.querySelector('.map-slider')`, `.builder-slider-left`, `.builder-slider-right`) came back `null` for all three: this particular page simply has no map section or builder-slider markup at all: the script is theme-wide boilerplate shared across every "easton"-family site, shipped even on pages that don't use it. The only reason it surfaced at all was a `jQuery is not defined` page error once jQuery's own CDN script tag (served from the site's own domain) was correctly stripped as CMS runtime. Rather than self-host a whole jQuery+Slick vendor stack to run a script with nothing to act on, the dead init script was simply dropped — same "verify the elements actually exist before building for them" lesson as Erin Ridge North's unreachable popup trigger, just at the level of an entire orphaned script block instead of one button.
- Also hit the now-familiar **bug 4** (`<picture><source type="image/webp">` 404s — see the Sherwood Golf Estates write-up above) on two images; fixed the same way, by stripping the orphaned `<source>` tags.
- Several below-the-fold builder-logo images only ever exist as literal text inside `<noscript>` fallback blocks (WP Rocket's standard lazy-load no-JS fallback), confirmed via a direct DOM query finding no corresponding live `<img>` element at all — inert to any JS-enabled browser, including this archival copy, exactly like a GTM/Facebook Pixel noscript tracking-pixel fallback. Left as-is; nothing to fetch.

Heartland Meadows was the most different stack yet on this project: WPBakery Page Builder instead of Elementor/Beaver Builder/a custom theme, Contact Form 7 instead of Gravity Forms, and Revolution Slider instead of Slick or Elementor's native Swiper widget — a new hero-carousel mechanism, each with its own quirks:

- **Revolution Slider renders each slide's text layers with runtime-computed, viewport-specific absolute-pixel inline styles**, kept in sync with window size by RevSlider's own JS (stripped as CMS runtime) — freezing the captured snapshot's exact pixel positions would look wrong at any other viewport. Inspection showed all 3 slides on this page share the exact same background photo and differ only in their text captions, so rather than reproduce RevSlider's pixel-perfect layer system, the hero was rebuilt as a single static background image with a small vanilla-JS crossfade of just the 3 text blocks, laid out with ordinary flex/flow CSS that reflows at any viewport — the same "rebuild as vanilla JS rather than self-host/reproduce the whole plugin" precedent as Ridgeview's Swiper widget and East Pine's gallery lightbox.
- **A corrupted-DOM bug from replacing only the slider's inner `<ul>` instead of its whole wrapper chain.** The widget is nested six levels deep (`wpb_wrapper` > `wpb_revslider_element` > `forcefullwidth_wrapper_tp_banner` > `..._wrapper` > `rev_slider_1_1` > the `<ul>` of slides), and every one of those wrapper divs carries RevSlider's own runtime-computed absolute-pixel inline styles. A first pass replaced only the innermost `<ul>` with the new static hero, leaving all 5 outer wrapper divs — and their non-matching `</div>` closes after a trailing `<!-- END REVOLUTION SLIDER -->` comment — stranded in the document. The browser's forgiving HTML parser papered over it well enough that Playwright still found 3 working slides and clickable bullets, but the imbalance was silently shifting every sibling section's DOM nesting afterward. Fixed by replacing the entire `wpb_wrapper`-through-"END REVOLUTION SLIDER"-plus-5-closing-divs span with just the clean hero div, keeping only the legitimate outer `vc_column-inner` column wrapper.
- **RevSlider's own helper functions (`setREVStartSize`, `revslider_showDoubleJqueryError`) live elsewhere in the document as theme/plugin boilerplate, not next to the widget markup**, and both call the global `jQuery` — stripped as CMS runtime, this threw a `jQuery is not defined` page error the instant the page loaded. Same category of bug as Westwinds' dead carousel-init script, just RevSlider's own library code instead of a theme's carousel init; fixed by dropping both dead script blocks outright, confirmed via Playwright's `pageerror` listener (0 errors after the fix, down from 1).
- Contact Form 7's reCAPTCHA v3 ("invisible") field was dropped from the rewritten form entirely (same reasoning as Heritage Valley Estates' v2 "bubble" shell), which left two reCAPTCHA loader `<script>` tags and a `.grecaptcha-badge` floating widget fixed to the bottom-right corner, both entirely orphaned — stripped outright, the badge's true matching close found via the project's `find_div_end` balanced-tag depth scanner rather than a guessed fixed-count `</div></div>` (a non-greedy regex attempt failed outright here, since the badge's real structure — a logo `<div>` containing an iframe, an error `<div>`, a hidden `<textarea>`, and a sibling hidden `<iframe>` — doesn't align with where a literal `</div></div>` first appears).
- `#mortgage_calc_section` and its surviving `mortgage-calculator-js`/`mortgage-calculator-js-extra` script tags are entirely inert on this particular page — a direct DOM query confirmed the section renders as nothing but a static lot-map image, with zero `<input>`/`<select>`/`<button>` elements anywhere in it. Left as dead CMS boilerplate, same "verify the elements actually exist" lesson as Westwinds' dead carousel init.

Fairways North (plain Elementor on "Hello Elementor", same theme family as East Pine) was a comparatively straightforward build, re-confirming two established patterns rather than surfacing new bugs:

- Elementor's native Swiper "Slides" widget hero, same duplicate-slide problem as Ridgeview at the Quarry and Heritage Valley Estates: the captured snapshot had 27 slide divs for 9 real slides (loop-mode clones in `swiper-slide-duplicate`/`-duplicate-prev`/`-duplicate-next`/`-duplicate-active` variants). Rebuilt as the same small vanilla-JS fade carousel used on those two sites, keeping each slide's unique `elementor-repeater-item-<hash>` class (its background image is set via a CSS rule scoped to that class).
- Two Gravity Forms using the newer "gravity-theme" fieldset markup (same as Heritage Valley Estates): a short newsletter-style signup (First/Last/Email only) and a full contact form (name, phone, email, message, plus a newsletter-consent checkbox). Both rewritten to POST JSON to `/api/lead`, each with its own status element and honeypot field.
- Elementor's native masonry gallery widget again used the `data-thumbnail` lazy-load pattern (bug 5), already fixed by the standard pre-bundling pre-pass.

East Village (a rental-apartment listing page, same category as East Pine) is the first Divi-theme site on this project — every prior site used Elementor, Beaver Builder, WPBakery, or a custom Foundation theme:

- **Divi's own JS-driven Google Maps module** (`.et_pb_map`, positioned via `data-center-lat`/`-lng`/`-zoom` attributes) is captured already-initialized, the DOM snapshot full of the live Google Maps JS API's own generated UI chrome (zoom buttons, keyboard-shortcut hints). The init code that reads those data attributes and actually builds the map lives in Divi's own frontend bundle, served from the site's own domain and correctly stripped as CMS runtime — left as-is, this would ship a dead, half-rendered map plus several now-pointless `maps.googleapis.com` script tags still being fetched for nothing. Replaced the whole module with a plain Google Maps iframe embed built directly from the data attributes already on the page (the same "legitimate third-party service, no tracking backend" pattern used for every other site's map), and dropped the now-unused Maps JS API script tags.
- **Mailchimp's own tracking/popup "connected site" loader** (`chimpstatic.com/mcjs-connected/...`) is a new tracker family for this project (previously only Facebook/GTM/GA/Clarity/HubSpot) — added permanently to `clean_site.py`'s strip lists. It's an inline script that builds its own `<script src>` tag at runtime rather than being a plain `<script src=...>` itself, so it needed a body-text substring match, not just a src match. The site's actual lead-capture form (Divi's "Email Optin" newsletter module) is unrelated to this tracking snippet and was rewritten normally to POST JSON to `/api/lead`.

## Deployment

Deployed to Vercel as a static project (no framework, no build step). Each site's URL is `<deployment>/<slug>/`.
