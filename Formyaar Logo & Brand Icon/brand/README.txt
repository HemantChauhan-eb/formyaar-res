FORMYAAR BRAND MARK — "Layered Lock-up" (3b)
================================================

The mark: an ink/white F sits behind an accent-blue Y sharing one
footprint — the "yaar" literally holding up the form. Colors match
the site's single-accent system (#305eff on ink #0c1322 / white).

FILES
-----
Vector masters (edit these first, re-export if you ever need other sizes):
  logo-mark-light-bg.svg   ink F + accent Y — use on white/light backgrounds
  logo-mark-dark-bg.svg    white F + accent Y — use on black/dark backgrounds
  app-icon.svg             solid accent-blue rounded square, white glyph —
                           use for app icons / avatars / anywhere you need
                           one flat tile that works on ANY background

Transparent PNGs (logo only, no background — for docs, decks, print, headers):
  logo-mark-light-bg-{16,32,64,128,192,256,512,1024}.png
  logo-mark-dark-bg-{16,32,64,128,192,256,512,1024}.png

App icon PNGs (solid blue tile, pre-sized for common platform slots):
  app-icon-1024.png   App Store / Play Store master
  app-icon-512.png    PWA manifest "512x512"
  app-icon-192.png    PWA manifest "192x192" / Android adaptive icon
  app-icon-180.png    Apple touch icon (apple-touch-icon.png)
  app-icon-167.png    iPad touch icon
  app-icon-152.png    iPad legacy touch icon
  app-icon-120.png    iPhone touch icon
  app-icon-48.png     Windows tile / desktop shortcut
  app-icon-32.png     favicon fallback
  app-icon-16.png     favicon fallback

favicon.ico
  Multi-resolution (16/32/48px) — drop straight into your site root
  as /favicon.ico, or reference: <link rel="icon" href="/favicon.ico">

QUICK HTML SETUP
----------------
<link rel="icon" href="/favicon.ico" sizes="any">
<link rel="icon" type="image/png" sizes="32x32" href="/app-icon-32.png">
<link rel="apple-touch-icon" sizes="180x180" href="/app-icon-180.png">
<link rel="icon" type="image/png" sizes="192x192" href="/app-icon-192.png">
<link rel="icon" type="image/png" sizes="512x512" href="/app-icon-512.png">

For the site header/nav (next to the wordmark), use logo-mark-light-bg.svg
at whatever height matches your text (e.g. 28–36px). If the header is a
dark section, swap to logo-mark-dark-bg.svg.
