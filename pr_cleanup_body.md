## Summary
Remove the custom-HTML/CSS ASCII-diagram wrapping from the whole blog (per direction), and drop
hero images. This cleans up after the merged site-fixes PR #12.

## Changes
- Delete `layouts/shortcodes/diagram.html`, `static/css/custom.css`,
  `layouts/partials/custom_head.html` — the diagram-wrapping pattern is gone.
- `content/posts/incus-home-lab-experience.md`: ASCII diagram converted to a plain fenced code
  block; hero image removed.
- Delete `static/images/incus-homelab-hero.png`.

## Kept on purpose
- `layouts/shortcodes/recent-posts.html` — this is a post-list shortcode that drives the dynamic
  Home "Recent posts" section. It is NOT diagram wrapping (it generates a plain `<ul>` and needs no
  custom CSS). Left in place so the home page still auto-lists new posts. Flag if you want it gone
  too (Home would then need a static list again).

## Future diagrams
Per direction: use Excalidraw / draw.io, export a PNG, and attach it to the post as a normal image
— no custom shortcodes or CSS.

## Verification
- `hugo --gc --minify` clean. No `ascii-diagram` class, no hero images, Home still lists posts.
