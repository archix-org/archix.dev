## Project

- The docs under `src/content/docs/docs/` describe the archix CLI in `archix-org/archix`. Quote its output, options and flags exactly as the code prints them, and keep them in step with it.
- The site is written as if archix v1 is released; the only pre-alpha notice is the banner in `src/components/Header.astro`.
- Lua examples must evaluate: a module with `imports`, `options` or `config` can't have other fields, `programs.*` needs `home.user` and a `users` entry, and host files are `lua/hosts/<host>/init.lua`.
- Never put personal or machine-specific details in committed files.

## Development

When starting the dev server, use background mode:

```
astro dev --background
```

Manage the background server with `astro dev stop`, `astro dev status`, and `astro dev logs`.

## Documentation

Full documentation: https://docs.astro.build

Consult these guides before working on related tasks:

- [Adding pages, dynamic routes, or middleware](https://docs.astro.build/en/guides/routing/)
- [Working with Astro components](https://docs.astro.build/en/basics/astro-components/)
- [Using React, Vue, Svelte, or other framework components](https://docs.astro.build/en/guides/framework-components/)
- [Adding or managing content](https://docs.astro.build/en/guides/content-collections/)
- [Adding styles or using Tailwind](https://docs.astro.build/en/guides/styling/)
- [Supporting multiple languages](https://docs.astro.build/en/guides/internationalization/)
