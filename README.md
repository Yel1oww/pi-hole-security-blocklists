# 🛡️ pi-hole-blocklists

Curated DNS blocklists for Pi-hole, AdGuard Home, and anything else that eats a
list of domains. Rebuilt weekly from five public threat intelligence feeds.

![Updated weekly](https://img.shields.io/badge/updated-weekly-blue)
![Automated](https://img.shields.io/badge/built%20with-n8n-EA4B71)

**This repo holds no files** — just the links and the explanation. The lists
themselves live in the repo that generates them, so you're always subscribing to
the freshest copy with no mirroring delay.

---

## The lists

### 🟢 Strict — recommended

Malware-dedicated infrastructure only: command-and-control servers, purpose-built
malware hosting, disposable domains registered to do harm. Conservative, with a
low false-positive risk.

```
https://raw.githubusercontent.com/Yel1oww/n8n-threat-blocklist/refs/heads/main/lists/strict.txt
```

Use this if you want to add a list and never think about it again.

### 🟠 Aggressive — wider coverage

Everything the pipeline collects: the strict list, plus phishing pages,
compromised legitimate sites, and historical malware domains.

```
https://raw.githubusercontent.com/Yel1oww/n8n-threat-blocklist/refs/heads/main/lists/aggressive.txt
```

Much broader protection, but it *will* occasionally block something you wanted.
Only use it if you're comfortable allowlisting a domain when a site breaks.

**Don't add both.** Strict is a subset of aggressive, so adding both just does
the same work twice.

---

## Which should I pick?

The difference comes down to one question: **was this domain built to be
malicious, or is it a legitimate site that got hacked?**

| | Strict | Aggressive |
|---|---|---|
| Malware C2 servers | ✅ | ✅ |
| Purpose-built malicious domains | ✅ | ✅ |
| Compromised legitimate sites | ❌ | ✅ |
| Phishing pages on hacked hosts | ❌ | ✅ |
| Historical malware domains | ❌ | ✅ |
| **False-positive risk** | **Low** | **Moderate** |

A hacked WordPress site really is serving malware today — and next week the owner
patches it and it's an ordinary bakery again. Aggressive blocks it; strict
doesn't, because permanently blocking a victim's website causes more harm than it
prevents.

Purpose-registered infrastructure has no such lifecycle. It was born bad and
stays bad. That's strict.

**Not sure? Start with strict.** You can always switch later.

---

## Adding a list

<details open>
<summary><b>Pi-hole (v5 and v6)</b></summary>

**Web UI:** Adlists → paste the URL → **Add**. Then Tools → Update Gravity.

**Command line:**

```bash
pihole -a adlist add https://raw.githubusercontent.com/Yel1oww/n8n-threat-blocklist/refs/heads/main/lists/strict.txt
pihole -g
```

Pi-hole updates gravity weekly by default, so it'll pick up new versions on its
own once added.
</details>

<details>
<summary><b>AdGuard Home</b></summary>

Filters → DNS blocklists → **Add blocklist** → Add a custom list → paste the URL.

Set the update interval to 24 hours under Settings → General.
</details>

<details>
<summary><b>pfBlockerNG (pfSense / OPNsense)</b></summary>

DNSBL → DNSBL Groups → Add. Set Format to **Auto**, State to **ON**, paste the
URL, and set the update frequency to Daily.
</details>

<details>
<summary><b>Blocky</b></summary>

```yaml
blocking:
  blackLists:
    malware:
      - https://raw.githubusercontent.com/Yel1oww/n8n-threat-blocklist/refs/heads/main/lists/strict.txt
  clientGroupsBlock:
    default:
      - malware
```
</details>

<details>
<summary><b>Unbound (RPZ-style)</b></summary>

The lists are plain domains, so convert before use:

```bash
curl -s https://raw.githubusercontent.com/Yel1oww/n8n-threat-blocklist/refs/heads/main/lists/strict.txt \
  | grep -v '^#' | grep -v '^$' \
  | awk '{print "local-zone: \""$1"\" always_nxdomain"}' \
  > /etc/unbound/unbound.conf.d/blocklist.conf
sudo unbound-control reload
```
</details>

<details>
<summary><b>Anything else</b></summary>

Plain text, one domain per line, `#` for comments. No IPs, no wildcards, no hosts
file prefixes. If your tool accepts a domain list over HTTPS, it'll work.
</details>

---

## Format

```
# Strict blocklist
# Generated 2026-08-21 09:48 UTC by n8n-threat-blocklist
# 422 domains
# Malware-dedicated infrastructure only. Conservative; safe to subscribe blind.
# Sources: abuse.ch (ThreatFox, URLhaus), blackbook, OTX, OpenPhish
# Filtered against the Tranco top domains. Report false positives via GitHub issues.

193-233-126-53.sslip.io
1hvnc.duckdns.org
222align.noip.at
...
```

Every file carries its generation timestamp and domain count in the header, so
you can always tell how fresh your copy is.

---

## How they're built

```
5 feeds  →  normalise  →  allowlist  →  score  →  2 lists  →  GitHub
 daily      URL→domain    Tranco       tier +     strict     weekly
                          top 100k     dedicated  aggressive  commit
```

**Collected daily, published weekly.** Feeds are polled every day into a local
database; the lists are regenerated and committed every Sunday. That cadence
means these are best understood as *durable malicious infrastructure* rather than
a fast-moving phishing feed — phishing domains often live only a day or two.

### Sources

| Feed | Contributes |
|---|---|
| [ThreatFox](https://threatfox.abuse.ch) | Live malware command-and-control infrastructure |
| [URLhaus](https://urlhaus.abuse.ch) | Malware distribution URLs |
| [blackbook](https://github.com/stamparm/blackbook) | Historical malware-associated domains |
| [OTX](https://otx.alienvault.com) | Community-submitted indicators |
| [OpenPhish](https://openphish.com) | Phishing URLs |

All credit for the underlying data goes to those projects. This is aggregation
and filtering, not original research.

### Safeguards

- **Popular domains are never blocked.** Everything is checked against the
  [Tranco](https://tranco-list.eu) top 100,000 before publication, whatever a
  feed claims about it.
- **Confirmed false positives are permanently allowlisted** and won't reappear
  even if a feed relists them.
- **Stale indicators expire.** Domains not re-observed within 90 days are
  dropped, so the lists describe current threats rather than accumulating
  forever.
- **IP addresses are discarded** — these are DNS blocklists, nothing else.

---

## Something's blocked that shouldn't be

Sorry — that's the trade-off with any blocklist.

**Fix it immediately:** allowlist the domain in your own DNS blocker. In Pi-hole
that's Domains → Add to Allowlist, and it takes effect straight away.

**Then tell me:** open an issue in the
[generator repo](https://github.com/Yel1oww/n8n-threat-blocklist/issues) with the
domain and why you think it's wrong. Confirmed false positives go on the
permanent allowlist.

Please check *which* list you're using first. If a domain only appears in
aggressive, that's often working as intended — aggressive deliberately includes
compromised legitimate sites. Strict is the one that should stay clean.

---

## Source and self-hosting

Everything — the aggregation script, both n8n workflows, and full setup
instructions — lives in
**[Yel1oww/n8n-threat-blocklist](https://github.com/Yel1oww/n8n-threat-blocklist)**.

The workflow files there are exportable and contain no credentials, so you can
run the whole pipeline yourself against your own repo if you'd rather not depend
on mine.

---

## Maintenance

Maintained by [@Yel1oww](https://github.com/Yel1oww). Collection and publication
are automated; false-positive reports are reviewed by hand.

**If the lists stop updating for more than two weeks, assume something has
broken.** Check the
[commit history](https://github.com/Yel1oww/n8n-threat-blocklist/commits/main/lists)
to see when they were last rebuilt — a list that has silently stopped updating is
worse than no list at all.

---

## Disclaimer

Provided as-is, with no warranty. These lists are derived from third-party feeds
and may contain errors. Test before deploying anywhere that matters. The upstream
feeds are free under fair-use terms — check their licensing if you're using them
commercially.
