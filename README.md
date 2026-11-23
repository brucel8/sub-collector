# sub-collector

[中文说明](README.zh-CN.md)

A two-piece toolkit that harvests free V2Ray / Clash subscriptions from GitHub, **actually speed-tests every candidate node**, and publishes a merged subscription containing only nodes verified working minutes before release.

The hard problem with free subscriptions is not finding nodes — it is that most of them are already dead: the port still answers, but the protocol behind it is long gone, so a plain "is it alive" check cannot tell. The merger loads all candidates into the [mihomo](https://github.com/MetaCubeX/mihomo) core and fetches a test page *through each node*; a node ships only if it returns a real response.

Pure Python 3 standard library, zero pip dependencies. If the mihomo binary is missing, the script downloads the latest build for your architecture from GitHub automatically.

## How it works

```
sub_collector.py                     sub_merge.py
─────────────────                    ─────────────────────────────
Search GitHub for repos updated  →   Fetch every subscription → parse nodes
in the last 7 days → pick the        (URI / base64 / Clash YAML) → dedupe by
"all / combined" subscription        (protocol + server + port + credential)
files → light validation →           → drop ss / ssr / socks5 / http →
out/all_links.txt                    TCP pre-filter → mihomo real test →
                                     sort by latency → per-server cap →
                                     out/merged-*
```

**Node type policy:** only six types are kept — `vmess`, `vless`, `trojan`, `hysteria`, `hysteria2`, `tuic`. `ss` and `ssr` are dropped at the URI entry point; `socks5` and `http` only ever arrive via Clash YAML sources and are rejected by the type whitelist.

## Quick start

```bash
git clone https://github.com/elvisl8/sub-collector.git
cd sub-collector

python3 sub_collector.py   # harvest → out/all_links.txt
python3 sub_merge.py       # merge + real test → out/merged-v2ray.txt, out/merged-clash.yaml
```

Serve the results with any static HTTP server:

```bash
cd out && python3 -m http.server 8351
# subscription URLs, e.g.:
#   http://<your-ip>:8351/merged-clash.yaml
#   http://<your-ip>:8351/merged-v2ray.txt
```

## Output files

| File | Contents |
|---|---|
| `out/latest.txt` | Harvest list: repo / last update / raw link / approx. node count |
| `out/all_links.txt` | Plain subscription link list (the merger's input) |
| `out/report-YYYY-MM-DD.md` | Daily harvest report |
| `out/merged-v2ray.txt` | Merged subscription, base64 (works in v2rayN and similar clients) |
| `out/merged-v2ray-plain.txt` | Same, plain text (for debugging) |
| `out/merged-clash.yaml` | Full Clash / mihomo config (nodes + ACL4SSR groups and rules) |
| `out/merge-stats.txt` | This run's merge statistics (the `Done:` line) |

## Scheduled runs

### Option A: GitHub Actions (hosted — nothing to run yourself)

The bundled workflow `.github/workflows/daily-collect.yml` runs the whole pipeline on GitHub once a day at **07:30 Asia/Shanghai** (23:30 UTC; GitHub cron can start a few minutes late). It runs the collector, then the merger, and commits `out/` and `state.json` back to the repo, so the subscription files are served straight from the repo's raw URLs:

- `https://raw.githubusercontent.com/elvisl8/sub-collector/main/out/merged-clash.yaml`
- `https://raw.githubusercontent.com/elvisl8/sub-collector/main/out/merged-v2ray.txt`

You can also start a run manually any time from the repo's Actions tab (`workflow_dispatch`). No secrets to configure — the workflow uses the built-in `GITHUB_TOKEN`.

### Option B: your own server (cron)

One round every 8 hours; collector and merger are chained (the merger must run after the collector):

```cron
CRON_TZ=Asia/Shanghai
20 */8 * * * /usr/bin/python3 /path/to/sub_collector.py >> /path/to/run.log 2>&1; /usr/bin/python3 /path/to/sub_merge.py >> /path/to/merge.log 2>&1
```

## Environment variables

**sub_collector.py**

| Variable | Default | Notes |
|---|---|---|
| `GITHUB_TOKEN` | empty | Raises the GitHub API limit from 60 to 5,000 requests/hour (optional) |
| `MAX_REPOS` | 60 | Max repos to scan in depth |
| `DAYS` | 7 | Only repos updated within this many days |
| `MAX_LINKS` | 120 | Max links in the output list |

**sub_merge.py**

| Variable | Default | Notes |
|---|---|---|
| `LINKS_FILE` | `/opt/sub-collector/out/all_links.txt` | Subscription list path; override with a relative path for local debugging |
| `OUT_DIR` | Same directory as `LINKS_FILE` | Output directory |
| `MAX_NODES` | 200 | Node cap per output file |
| `ACL4SSR_INI` | gtang8/ACL4SSR fork, Mini config | Clash rules ini URL; can point to the official ACL4SSR repo or your own fork |
| `CHECK_ALIVE` | 1 | Stage-1 TCP pre-filter switch (0 = off) |
| `CHECK_TIMEOUT` / `CHECK_WORKERS` | 3 s / 100 | Pre-filter timeout and concurrency |
| `REAL_TEST` | 1 | Stage-2 real test switch (0 = off, falls back to stage-1 only) |
| `REAL_CAP` | 4000 | Max candidates to real-test |
| `REAL_CHUNK` / `REAL_WORKERS` | 400 / 40 | Nodes per mihomo test chunk / parallel test workers |
| `REAL_TIMEOUT_MS` | 6000 | Per-node test timeout (ms) |
| `REAL_DEADLINE` | 900 | Overall real-test budget (seconds); ships with whatever has passed when it expires |
| `PER_IP_CAP` | 3 | Max nodes per server in the final output |
| `MIHOMO_BIN` | `mihomo` next to the script | mihomo binary path |

## Reading the merge statistics

A real run (2026-11-23):

```
Done: sources=86 v2ray_nodes=200 clash_proxies=200 dup_removed=48666
      ss_dropped=10079 bad_skipped=586 clash_bad=1 endpoints=13007
      alive_endpoints=7633 dead_dropped=5859 unchecked_udp=1235
      real_tested=3198 real_passed=371 real_bad=2
```

- `dup_removed` — duplicates removed by (protocol + server + port + credential); the same node under a different name still counts once
- `ss_dropped` — `ss` **and** `ssr` entries dropped at the entry point (both types share this counter)
- `bad_skipped` / `clash_bad` — entries rejected as malformed (illegal port, bad UUID format, placeholder server, sample templates mixed into upstream repos, …)
- `real_tested / real_passed / real_bad` — real-test totals: tested / passed / poison entries rejected by the core

## Design notes

- **Two-stage filtering:** the TCP pre-filter kills whole batches of dead endpoints (each endpoint is probed only once); the real test then verifies protocol-level usability node by node. UDP protocols such as hysteria2 / tuic are really tested too, not waved through on an open port.
- **YAML sources are never passed through verbatim:** every Clash YAML entry is parsed into fields, rebuilt from a per-type field whitelist, and validated (type / server / port / UUID format …) before it can ship — the sample templates, placeholder entries and broken indentation mixed into free repos are stopped here (a single bad entry can make a whole config fail to import).
- **Poison-entry isolation:** one malformed entry can make mihomo refuse to load an entire test config, so failing chunks are bisected until the bad entry is isolated and dropped alone, without taking the good nodes in its batch down with it.
- **Source fairness:** candidates are interleaved round-robin across sources before the cap is applied, so the first subscriptions in the list cannot eat the whole quota.
- **ACL4SSR rules:** the ini template is fetched at runtime to generate proxy groups and rule-providers; if the fetch fails, the build falls back to a single select group + MATCH instead of failing.
- **Type policy:** `ss`, `ssr`, `socks5` and `http` are always dropped; only `vmess`, `vless`, `trojan`, `hysteria`, `hysteria2` and `tuic` ship.

## Notes

- Without `GITHUB_TOKEN`, the GitHub core API limit is 60 requests/hour; the collector stops scanning when it hits the limit and still publishes a list from what it has, instead of wasting the whole round. A `state.json` cache lets unchanged repos reuse their previous results to save quota.
- Free nodes die fast by nature: this tool guarantees "verified working at publish time", and a batch will still die off within hours. Just refresh.
- For learning and technical research only; please comply with local laws and regulations.

## License

MIT
