# docs.sim.compute.cx

Customer documentation for [tsim](https://sim.compute.cx) by Theoric Labs. Hosted by [Mintlify](https://mintlify.com) at `https://docs.sim.compute.cx`.

This repository is **docs only** (mirroring [`theoriclabs/docs.compute.cx`](https://github.com/theoriclabs/docs.compute.cx)).

## Preview & Validate

```bash
npx mint dev
npx mint validate
```

## Connect Mintlify Custom Domain

1. Connect `theoriclabs/docs.sim.compute.cx` in the Mintlify dashboard (or run `npx mint deploy --github --subdomain tsim-docs`).
2. Attach custom domain `docs.sim.compute.cx` in Mintlify (`cname.mintlify.builders.` + ACME TXT records).
3. Apply the DNS records via `namecheap` CLI on `compute.cx`.
