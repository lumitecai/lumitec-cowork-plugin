# Marketplace IDs & Region Endpoints

## Region Endpoints

Use region aliases, NOT AWS region codes:

| Region | Alias | Endpoint |
|---|---|---|
| North America | `na` | sellingpartnerapi-na.amazon.com |
| Europe | `eu` | sellingpartnerapi-eu.amazon.com |
| Far East | `fe` | sellingpartnerapi-fe.amazon.com |

## Marketplace IDs

### North America (region: na)
| Country | Code | Marketplace ID |
|---|---|---|
| United States | US | ATVPDKIKX0DER |
| Canada | CA | A2EUQ1WTGCTBG2 |
| Mexico | MX | A1AM78C64UM0Y8 |

### Europe (region: eu)
| Country | Code | Marketplace ID |
|---|---|---|
| United Kingdom | UK | A1F83G8C2ARO7P |
| Germany | DE | A1PA6795UKMFR9 |
| France | FR | A13V1IB3VIYZZH |
| Italy | IT | APJ6JRA9NG5V4 |
| Spain | ES | A1RKKUPIHCS9HS |
| Ireland | IE | A28R8C7NBKEWEA |
| Netherlands | NL | A1805IZSGTT6HS |
| Sweden | SE | A2NODRKZP88ZB9 |
| Poland | PL | A1C3SOZRARQ6R3 |
| Belgium | BE | AMEN7PMS3EDWL |
| Saudi Arabia | SA | A17E79C6D8DWNP |
| UAE | AE | A2VIGQ35RCS4UG |
| Egypt | EG | ARBP9OOSHTCHU |
| Turkey | TR | A33AVAJ2PDY3EV |
| South Africa | ZA | AE08WJ6YKNBMC |

### Far East (region: fe)
| Country | Code | Marketplace ID |
|---|---|---|
| Japan | JP | A1VC38T7YXB528 |
| Australia | AU | A39IBJ37TRP1C6 |
| India | IN | A21TJRUUN4KGV |
| Singapore | SG | A19VAU5U5O7RUS |

## Critical Notes

- **Always pass marketplaceIds explicitly** — never rely on defaults
- EU API defaults to Germany. UK queries silently return nothing without explicit UK marketplace ID.
- Seller ID is per-region (one for NA, one for EU), not per-marketplace
- Get your seller IDs from `getMarketplaceParticipations`
