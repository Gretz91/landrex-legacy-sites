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
```

`index.html` references `style.css` and `assets/` with paths relative to its own folder, so each site is self-contained and can be opened directly or served from any static host.

## Sites

| Slug | Original domain | Status |
|---|---|---|
| tanglewood-ridge | tanglewoodridge.com | ✅ done, verified against live site |
| erin-ridge-north | erinridgenorth.ca | pending |
| ridgeview-at-the-quarry | ridgeviewatthequarry.ca | pending |
| windsor-pointe | windsorpointe.ca | pending |
| sherwood-golf-estates | sherwoodestates.ca | pending |
| east-pine | eastpine.ca | pending |
| heritage-valley-estates | heritagevalleyestates.ca | pending |
| westwinds | westwindsliving.ca | pending |
| heartland-meadows | theheartlandmeadows.com | pending |
| fairways-north | fairwaysnorth.com | pending |
| east-village | east-village.ca | pending |
| urban-district | urbandistrict.ca | pending |
| mirigal | mirigal.ca | pending |

Excluded: `livefenwyck.com` (owned by Avillia Developments, not Landrex), `mail.theheartlandmeadows.com` (email only), `forestheightsbeaumont.landrex.com` (unreachable/broken, not a live site).

## Build method

Documented in full in the Landrex project's `claude/legacy-sites-status.md` doc. Short version per site:

1. Capture the live page's DOM + only the CSS rules it actually uses ("critical CSS extraction"), self-host its images/fonts, and package as a bundle.
2. Strip every script tied to the old CMS: Facebook Pixel, GTM/GA, jQuery, Elementor runtime, Gravity Forms. Remove the now-dead `<link>` tags pointing at the live wp-content assets.
3. Re-add the `<link rel="stylesheet" href="style.css">` the bundler doesn't insert automatically.
4. Rewrite the contact form to post JSON to `https://landrex-web.vercel.app/api/lead` with a honeypot field, instead of the old Gravity Forms action.
5. Restore any working gallery/carousel with minimal vanilla JS (full Elementor/Swiper runtime isn't needed unless the page has a real multi-image carousel).
6. Verify: no tracking/CMS leftovers, HTML parses cleanly, screenshot comparison against the live site, test-submit the form with the honeypot filled.

## Deployment

Deployed to Vercel as a static project (no framework, no build step). Each site's URL is `<deployment>/<slug>/`.
