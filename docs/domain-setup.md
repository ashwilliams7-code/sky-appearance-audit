# Aurii domain setup

Primary website domain: `aurii.me`

Current status: repo-side brand/logo work is ready, but DNS activation requires a GoDaddy sign-in. Do **not** push a root `CNAME` before GoDaddy DNS is pointed at GitHub Pages, because GitHub may redirect the existing `github.io` Pages URL to `aurii.me` while `aurii.me` is still serving GoDaddy's default/parking site.

## Activation order

1. Update GoDaddy DNS for `aurii.me` using the records below.
2. Wait until `dig +short A aurii.me` returns GitHub Pages IPs.
3. Add a root `CNAME` file containing exactly:

```txt
aurii.me
```

4. Commit/push the `CNAME` file.
5. In GitHub Pages settings, verify custom domain `aurii.me` and enable HTTPS.
6. Forward the other purchased domains to `https://aurii.me/`.

## GoDaddy DNS records for `aurii.me`

Set these records in GoDaddy DNS for the apex/root domain:

- `A` / Host `@` / Value `185.199.108.153` / TTL default
- `A` / Host `@` / Value `185.199.109.153` / TTL default
- `A` / Host `@` / Value `185.199.110.153` / TTL default
- `A` / Host `@` / Value `185.199.111.153` / TTL default
- `CNAME` / Host `www` / Value `ashwilliams7-code.github.io` / TTL default

Optional IPv6 records:

- `AAAA` / Host `@` / Value `2606:50c0:8000::153`
- `AAAA` / Host `@` / Value `2606:50c0:8001::153`
- `AAAA` / Host `@` / Value `2606:50c0:8002::153`
- `AAAA` / Host `@` / Value `2606:50c0:8003::153`

## Redirect the other purchased domains

Use GoDaddy domain forwarding with permanent redirect to:

`https://aurii.me/`

Forward these domains:

- `aurii.info`
- `aurii.shop`
- `aurii.xyz`
- `aurii.pro`
- `aurii.vip`

Recommended forwarding settings:

- Type: Permanent / 301
- Destination: `https://aurii.me/`
- Path forwarding: enabled if GoDaddy offers it
- Masking: off

## Verification commands

```bash
dig +short A aurii.me
dig +short CNAME www.aurii.me
curl -I https://aurii.me/
```

Expected `A` answers are the four GitHub Pages IPs above. `curl -I` should eventually return a 200/301/302 from GitHub Pages with HTTPS working.
