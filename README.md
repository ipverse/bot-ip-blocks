# bot-ip-blocks

Daily-updated IP prefix lists for well-known **web crawlers** and **uptime monitoring probes**.
The data comes straight from each provider's own published IP list, gets normalized, and is collapsed down to the smallest set of CIDR blocks that still covers the same addresses.
No APIs, no databases, just files you can `curl`.

The main reason this exists: powering the crawler / monitoring badge on [lens.ipverse.net](https://lens.ipverse.net) when you look up an address.

Two datasets:

- **`crawlers.json`** for search engines, SEO bots, and AI/LLM crawlers (Googlebot, Bingbot, Ahrefs, GPTBot, ClaudeBot, PerplexityBot, Yandex, etc.)
- **`monitoring.json`** for third-party uptime and synthetic monitoring probes (Pingdom, UptimeRobot, Datadog, New Relic, StatusCake, Better Stack, updown.io)

Each service entry includes the IPv4 and IPv6 prefixes plus the things you actually need to recognize the bot from a request: glob patterns for the User-Agent header and, where the provider supports it, reverse-DNS hostname patterns. Git history lets you track how each service's ranges drift over time.

A human-readable rollup of service counts and totals lives in [STATS.md](STATS.md).

## JSON format

Both files have the same shape:

```json
{
  "services": {
    "Googlebot": {
      "description": "Google's primary search indexing crawler.",
      "source_url": "https://developers.google.com/search/apis/ipranges/googlebot.json",
      "user_agent_patterns": ["*Googlebot*", "*Googlebot-Image*"],
      "rdns_patterns": ["crawl-*.googlebot.com", "*.google.com"],
      "ip_list_authoritative": true,
      "ipv4": ["66.249.64.0/20"],
      "ipv6": ["2001:4860:4801::/64"]
    }
  }
}
```

A few things worth knowing:

- Prefixes are aggregated. Adjacent and overlapping CIDR blocks get merged into their parent, single IPs are promoted to `/32` or `/128`, and the result is the smallest CIDR set covering the same address space.
- `user_agent_patterns` and `rdns_patterns` are glob patterns (`*` wildcard), not literal strings. Match them against the request UA header or the reverse DNS name with any standard glob library.
- `ip_list_authoritative` tells you whether IP membership alone is a reliable identifier for the bot. Most services use crawler-specific lists published upstream and are `true`. A handful — currently Meta-ExternalAgent and Yandex — use the provider's full ASN aggregate, which covers non-crawler traffic (mail, ads, cloud, edge). For those, treat the IP list as a prefilter and confirm the match with `user_agent_patterns` and a forward-confirmed reverse-DNS check against `rdns_patterns` before acting on it.
- Each commit in this repo is a refresh, so `git log` is the canonical timeline.
- If an upstream fetch fails, the service is still emitted with empty arrays, so the shape of the document stays stable. STATS.md is the place to look for which service came up empty.

## How to use

Crawler list:
```bash
curl https://raw.githubusercontent.com/ipverse/bot-ip-blocks/master/crawlers.json
```

Monitoring list:
```bash
curl https://raw.githubusercontent.com/ipverse/bot-ip-blocks/master/monitoring.json
```

## Use cases

- Allowlist real search and AI crawlers at the WAF or CDN edge
- Allowlist uptime probes so they stop tripping rate limits and bot mitigation
- Figure out which bot is hitting you from request logs (by IP, user-agent, or rDNS)
- Catch spoofed user-agents by checking the source IP against the service's actual ranges
- Bot-traffic analysis and general network research

## Related projects

- **[as-ip-blocks](https://github.com/ipverse/as-ip-blocks)**: Daily-updated announced prefixes for every actively-announcing autonomous system.
- **[as-metadata](https://github.com/ipverse/as-metadata)**: Full AS metadata with categories, network roles, and historical fields.

## Questions or issues?

Head over to the [feedback repository](https://github.com/ipverse/feedback) if you have questions, issues, or suggestions.

## License

This data is released under [CC0 1.0 Universal](LICENSE).
