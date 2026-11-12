# Soft Sensor Toolbox — landing page

Static marketing site for [softsensortoolbox.online](https://softsensortoolbox.online).
Plain HTML/CSS/JS — no build step, no backend, no Supabase, no forms or trackers.

- `index.html` — page content (the “Launch the toolbox” buttons link to the live app)
- `styles.css` — all styling
- `script.js` — mobile nav, FAQ accordion, scroll reveal, footer year
- `favicon.svg`

## Deploy (Vercel)

```bash
npx vercel deploy --prod   # run from this folder; project: soft-sensor-toolbox
```

Custom domain: `softsensortoolbox.online` and `www.softsensortoolbox.online` are
attached to the Vercel project. At Namecheap (Advanced DNS) the records are:

| Host | Type | Value |
| --- | --- | --- |
| `@` | A | `216.198.79.1` |
| `@` | A | `64.29.17.1` |
| `www` | CNAME | `1a7109d9d3eab0d0.vercel-dns-017.com` |

Edit the files, run the deploy command again, and the site updates. To change
where “Launch the toolbox” points, search `index.html` for
`nadgawd.github.io/SoftSensorTB`.
