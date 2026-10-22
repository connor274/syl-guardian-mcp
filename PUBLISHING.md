# Publishing to the MCP registry (maintainers)

Namespace `io.sylai/guardian` is verified by proof of control over `sylai.io`, so it needs
DNS (or HTTP) authentication rather than GitHub login. Source: the registry's authentication doc
(verified 2026-10-22).

1. Generate an Ed25519 keypair (needs OpenSSL 3.0+; macOS's default LibreSSL does not
   support it, so use `brew install openssl` and call it by full path). Keep the private
   key out of any repo:
   ```bash
   openssl genpkey -algorithm Ed25519 -out key.pem
   PUBLIC_KEY="$(openssl pkey -in key.pem -pubout -outform DER | tail -c 32 | base64)"
   ```
2. Add a TXT record at the **apex** of `sylai.io` (not under a selector like `_mcp-auth`)
   in Cloudflare DNS, and wait for it to propagate:
   ```
   sylai.io. IN TXT "v=MCPv1; k=ed25519; p=<PUBLIC_KEY>"
   ```
3. Publish:
   ```bash
   brew install mcp-publisher
   mcp-publisher validate
   PRIVATE_KEY="$(openssl pkey -in key.pem -outform DER | tail -c 32 | xxd -p -c 64)"
   mcp-publisher login dns --domain sylai.io --private-key "${PRIVATE_KEY}"
   mcp-publisher publish
   curl "https://registry.modelcontextprotocol.io/v0.1/servers?search=io.sylai/guardian"
   ```
4. On each release, bump `version` here and in the private `syl_guardian` repo
   (`pyproject.toml`, `guardian/app.py`), then re-run `publish`.

Then follow the Smithery and Glama steps in the private repo's
`docs/mcp-registry-listing.md`.
