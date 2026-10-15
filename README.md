# archix.dev

The website and documentation for [archix](https://github.com/archix-org/archix), built with [Astro](https://astro.build) and [Starlight](https://starlight.astro.build).

```sh
npm ci
npm run dev      # http://localhost:4321
npm run build    # static output in dist/
```

- Landing page: `src/pages/index.astro` and `src/components/`.
- Docs: `src/content/docs/docs/`, served under `/docs/`. They describe archix as it behaves in `archix-org/archix`: a change there to behaviour, the CLI, config syntax or options needs a matching PR here.
- Brand assets come from the archix logo set; colours and fonts are defined in `src/styles/site.css` (landing) and `src/styles/starlight.css` (docs).

Merges to `main` build `ghcr.io/archix-org/site` and deploy it automatically.
