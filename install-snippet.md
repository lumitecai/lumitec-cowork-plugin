# Lumitec Amazon MCP — manual install for Claude Desktop / Cowork

If you're using Claude Desktop (or a Claude client that reads the same config file, like Cowork), paste the block below into your `mcpServers` section and restart the app.

**Config file location:**
- **macOS Desktop:** `~/Library/Application Support/Claude/claude_desktop_config.json`
- **Windows Desktop:** `%APPDATA%\Claude\claude_desktop_config.json`
- **Linux Desktop:** `~/.config/Claude/claude_desktop_config.json`
- **Cowork:** (same file path — Cowork shares Desktop's config)

**Replace every `REPLACE_…` placeholder with your actual values.** All values go in literal — no `${VAR}` expansion is performed by Desktop clients.

```jsonc
{
  "mcpServers": {
    "amazon-sp-api": {
      "type": "http",
      "url": "https://lumitec-sp-api-mcp.fly.dev/mcp",
      "headers": {
        "X-Lumitec-Key":           "REPLACE_WITH_LUMITEC_KEY",
        "X-SP-API-Client-Id":      "REPLACE_WITH_SP_API_CLIENT_ID",
        "X-SP-API-Client-Secret":  "REPLACE_WITH_SP_API_CLIENT_SECRET",
        "X-SP-API-Refresh-Token":  "REPLACE_WITH_SP_API_REFRESH_TOKEN",
        "X-SP-API-Region":         "eu",
        "X-SP-API-Marketplace-Id": "REPLACE_WITH_MARKETPLACE_ID",
        "X-SP-API-Seller-Id":      "REPLACE_WITH_SELLER_ID"
      }
    },
    "amazon-ads-api": {
      "type": "http",
      "url": "https://lumitec-ads-api-mcp.fly.dev/mcp",
      "headers": {
        "X-Lumitec-Key":       "REPLACE_WITH_LUMITEC_KEY",
        "X-Ads-Client-Id":     "REPLACE_WITH_ADS_API_CLIENT_ID",
        "X-Ads-Client-Secret": "REPLACE_WITH_ADS_API_CLIENT_SECRET",
        "X-Ads-Refresh-Token": "REPLACE_WITH_ADS_API_REFRESH_TOKEN",
        "X-Ads-Region":        "eu"
      }
    }
  }
}
```

## Getting your values

| Placeholder | Where to get it |
|---|---|
| `REPLACE_WITH_LUMITEC_KEY` | Issued by Lumitec support — email support@lumitec.ai |
| `REPLACE_WITH_SP_API_CLIENT_ID` / `_CLIENT_SECRET` | From your Amazon Developer Console SP-API app |
| `REPLACE_WITH_SP_API_REFRESH_TOKEN` | Output of Amazon's seller-authorization flow (region-specific) |
| `REPLACE_WITH_MARKETPLACE_ID` | Your primary Amazon marketplace ID, e.g. `A1F83G8C2ARO7P` (UK) |
| `REPLACE_WITH_SELLER_ID` | Seller Central → Settings → Account Info → Merchant Token |
| `REPLACE_WITH_ADS_API_*` | Separate Amazon Advertising API app credentials |

## Verifying

After restarting Claude, ask:
- *"list my amazon marketplaces via the sp-api connector"* — should return your active marketplaces
- *"list my amazon ads profiles"* — should return your ad accounts

If either errors with "missing or invalid X-Lumitec-Key", double-check you pasted the Lumitec key correctly into both MCP entries.
