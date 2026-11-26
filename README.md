# bot-ip-blocks

IP prefix lists for well-known **web crawlers** and **uptime monitoring probes**.
The data comes straight from each provider's own published IP list, gets normalized, and is collapsed down to the smallest set of CIDR blocks that still covers the same addresses.
No APIs, no databases, just files you can `curl`.

The main reason this exists: powering the crawler / monitoring badge on [lens.ipverse.net](https://lens.ipverse.net) when you look up an address.

Two datasets:

- **`crawlers.json`** for search engines, SEO bots, and AI/LLM crawlers (Googlebot, Bingbot, Ahrefs, GPTBot, ClaudeBot, PerplexityBot, Yandex, etc.)
- **`monitoring.json`** for third-party uptime and synthetic monitoring probes (Pingdom, UptimeRobot, Datadog, New Relic, StatusCake, Better Stack, updown.io)

Each service entry includes the IPv4 and IPv6 prefixes plus the things you actually need to recognize the bot from a request: glob patterns for the User-Agent header and, where the provider supports it, reverse-DNS hostname patterns. Each service also lists its User-Agent tokens with the purpose the vendor declares for each, so AI training crawlers, AI search crawlers and user-triggered AI fetchers can be told apart from classic search crawlers. A per-service `last_changed_at` timestamp tells you when the IP list last changed.

A human-readable rollup of service counts and totals lives in [STATS.md](STATS.md).

## Update notes

- **2026-11-26**: Every service now lists its user-agent tokens under `agents`, each with the purpose its vendor declares (search, AI training, AI search, user-triggered AI fetch, …). In `user_agent_patterns`, Applebot drops `*Applebot-Extended*` (a robots.txt-only token Apple says never crawls) and Googlebot drops `*Googlebot-News*` (a robots.txt-only token, Google News is crawled with the regular Googlebot user agents); Google (user-triggered) has `*GoogleProducer*`, the token Google Publisher Center's fetcher sends, in place of `*Google Publisher Center*`; New Relic drops `*Synthetics*`, which also matched Datadog's user agents (New Relic names no token for its synthetic monitors, they identify as Google Chrome)
- **2026-11-23**: Added the crawlers Google-Agent, OAI-AdsBot, MistralAI, MojeekBot, Amzn-SearchBot and Amzn-User, and the monitoring probes Sentry, Site24x7 and OneUptime; Google (user-triggered) now includes the previously missing `user-triggered-fetchers-google.json` ranges and the fetcher user agents Google currently documents; Amazonbot no longer includes a stray address picked up from the source page's analytics script

## JSON format

Both files have the same shape:

```json
{
  "services": {
    "ClaudeBot": {
      "description": "Anthropic's web crawlers (ClaudeBot, Claude-User, Claude-SearchBot)",
      "website": "https://www.anthropic.com",
      "source_url": "https://claude.com/crawling/bots.json",
      "last_changed_at": "2026-04-10T12:00:00Z",
      "user_agent_patterns": ["*ClaudeBot*", "*Claude-User*", "*Claude-SearchBot*"],
      "agents": [
        {"name": "ClaudeBot", "pattern": "*ClaudeBot*", "purpose": "ai-training"},
        {"name": "Claude-User", "pattern": "*Claude-User*", "purpose": "ai-assistant"},
        {"name": "Claude-SearchBot", "pattern": "*Claude-SearchBot*", "purpose": "ai-search"}
      ],
      "rdns_patterns": [],
      "ip_list_authoritative": true,
      "ipv4": ["192.0.2.0/24"],
      "ipv6": ["2001:db8::/32"]
    }
  }
}
```

A few things worth knowing:

- Prefixes are aggregated. Adjacent and overlapping CIDR blocks get merged into their parent, single IPs are promoted to `/32` or `/128`, and the result is the smallest CIDR set covering the same address space.
- `user_agent_patterns` and `rdns_patterns` are glob patterns (`*` wildcard), not literal strings. Match them against the request UA header or the reverse DNS name with any standard glob library.
- `agents` lists the service's user-agent tokens, one per entry of `user_agent_patterns` and in the same order: `name` is the vendor's token, `pattern` the glob from `user_agent_patterns`, `purpose` what the vendor declares that token is for. Always present; `[]` for a service identified by IP alone (currently OneUptime). Agent names are unique across both files. An agent is a token sent in the User-Agent header: a robots.txt-only token (`Applebot-Extended`, `Google-Extended`, `Googlebot-News`) is none.
- A User-Agent can match several agents of one service (`*Googlebot*` also matches `Googlebot-Image`, `*AdsBot-Google*` matches `AdsBot-Google-Mobile`): the agent with the longest pattern, counted without its `*`, wins.
- `purpose` is the vendor's own declaration per token, taken from its bot documentation, never a guess; without one it is `other`. A retired token keeps its last declaration. One purpose per token: where the vendor declares several uses for it, hedged ones ("may also be used to …") included, a declared AI use wins, in the order `ai-training`, `ai-search`, `ai-assistant` (Amazonbot is `ai-training`: its data "may be used to train Amazon AI models"). Except where the vendor offers a separate robots.txt control token for that AI use: a site controls the AI use through that token, so the crawler keeps the purpose of its other declared uses. With no AI use counted, the purpose is the one all of them fall under, else `other` (AhrefsBot: SEO platform and search engine). The vocabulary is closed:

  | `purpose`      | Meaning                                                                    |
  |----------------|----------------------------------------------------------------------------|
  | `search`       | A classic search engine: indexing, and fetching what its results show (favicons, page snapshots) |
  | `ai-training`  | Crawls to train models                                                     |
  | `ai-search`    | Indexes for an AI answer or search product                                 |
  | `ai-assistant` | Fetches on a user's request in an AI product (assistant, agent, AI answer) |
  | `seo`          | SEO and backlink tools                                                     |
  | `ads`          | Advertising: ad quality and landing-page checks, ad matching               |
  | `social`       | Link previews                                                              |
  | `monitoring`   | Uptime and synthetic checks                                                |
  | `other`        | Declared but none of these, several non-AI uses under different purposes, or undeclared; never a declared AI use without a control token, an undeclared token's use is unknown |

  A purpose's group is the part before the first `-`, so every `ai-*` purpose is "AI".

  Crawlers whose AI use is controlled by a robots.txt token, which a filter on `ai-training` or any other `ai-*` purpose does not catch:

  | Crawler (`purpose`)                    | Control token                                  | AI use it controls                                  |
  |----------------------------------------|------------------------------------------------|-----------------------------------------------------|
  | Googlebot (`search`), GoogleOther (`other`) | `Google-Extended`                         | Training and grounding for Google's generative AI   |
  | Applebot (`search`)                    | `Applebot-Extended`                            | Training of Apple's foundation models               |
  | YandexBot (`search`)                   | `YandexAdditional` (or `YandexAdditionalBot`)  | Search with Yandex AI responses                     |

  The control tokens are not agents: a site names them in robots.txt.

  Several vendors publish one IP list for tokens with different purposes (Anthropic, Perplexity, Mistral, Meta, …): for such a service the IP list alone cannot tell its agents apart, only the request's User-Agent can.
- `ip_list_authoritative` tells you whether IP membership alone is a reliable identifier for the bot. Most services use crawler-specific lists published upstream and are `true`. A handful — currently Meta-ExternalAgent and Yandex — use the provider's full ASN aggregate, which covers non-crawler traffic (mail, ads, cloud, edge). For those, treat the IP list as a prefilter and confirm the match with `user_agent_patterns` and a forward-confirmed reverse-DNS check against `rdns_patterns` before acting on it.
- `last_changed_at` is set per service and records when the service's IP list last changed (RFC 3339 UTC). If the upstream IPs haven't moved, the timestamp stays the same across refreshes; a change of `agents` or a purpose does not move it either.
- If an upstream source fails, the service keeps its previous IP list (and its `last_changed_at`) until the source recovers, so a failure never empties a service.

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
- Tell AI training crawlers, AI search crawlers and user-triggered AI fetchers from classic search crawlers by `purpose`
- Catch spoofed user-agents by checking the source IP against the service's actual ranges
- Bot-traffic analysis and general network research

## Related projects

- **[as-ip-blocks](https://github.com/ipverse/as-ip-blocks)**: Daily-updated announced prefixes for every actively-announcing autonomous system.
- **[as-metadata](https://github.com/ipverse/as-metadata)**: Full AS metadata with categories, network roles, and historical fields.

## Questions or issues?

Head over to the [feedback repository](https://github.com/ipverse/feedback) if you have questions, issues, or suggestions.
