# email-dns-frontend

Static React frontend for the Email DNS Inspector.

## Backend response contract for new panels

The frontend now renders two additional optional sections from the existing `POST /api/check` JSON response. Browser JavaScript cannot reliably perform authoritative DNS NS lookups or RDAP/WHOIS lookups by itself, so the backend should add these fields to the response.

### Live name servers

Return one of `nameServers`, `nameservers`, or `ns`:

```json
{
  "nameServers": {
    "status": "PASS",
    "checkedAt": "2026-06-09T00:00:00.000Z",
    "provider": {
      "name": "Cloudflare",
      "score": 95,
      "confidence": "high",
      "matchedBy": "*.cloudflare.com"
    },
    "records": [
      {
        "host": "ada.ns.cloudflare.com",
        "provider": { "name": "Cloudflare", "score": 95 }
      }
    ],
    "issues": []
  }
}
```

Backend implementation notes:

- Resolve live NS records for the inspected domain using DNS (`NS` query).
- Normalize hostnames to lowercase and strip trailing dots.
- Detect the DNS provider from the NS hostname suffix (for example Cloudflare, AWS Route 53, Google Cloud DNS, Azure DNS, GoDaddy, Namecheap, DNS Made Easy, NS1, Akamai, DigitalOcean, Hetzner, etc.).
- Include a provider `score` out of 100. This can be a static provider reputation/reliability score map at first, then later calculated from provider metadata or health checks.
- Return `WARN` if no NS records are found and `FAIL` only when the lookup errors in a way users should act on.

### Domain registrar

Return one of `registrar`, `domainRegistrar`, or `whois`:

```json
{
  "registrar": {
    "status": "PASS",
    "name": "Cloudflare, Inc.",
    "ianaId": "1910",
    "whoisServer": "whois.cloudflare.com",
    "url": "https://www.cloudflare.com",
    "createdAt": "2009-02-17T00:00:00.000Z",
    "updatedAt": "2026-01-10T00:00:00.000Z",
    "expiresAt": "2027-02-17T00:00:00.000Z",
    "domainStatus": ["clientTransferProhibited"],
    "source": "RDAP",
    "issues": []
  }
}
```

Backend implementation notes:

- Prefer RDAP for registrar data because it is structured and works better than scraping WHOIS text. Use WHOIS as a fallback where RDAP data is missing.
- Extract registrar name, IANA/registrar ID, WHOIS server, registrar URL, created/updated/expiry dates, and domain status values.
- Return `WARN` when registrar data is unavailable for unsupported TLDs or privacy-limited records, and include a human-readable issue.
- Keep CORS unchanged for the frontend origin and continue returning the fields from `POST /api/check` so the UI can render everything in a single analysis result.
