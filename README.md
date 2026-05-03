# Chimera Strategy Plugin — May 2026

The current live Chimera ruleset, captured as a single JSON plugin file.
Drop it into the `plugins/` folder of either tool to make it active:

* **Backtest Tool** (`chimeracloud/backtest-tool`) — runs the plugin against
  historic Betfair data and reports per-market P&L.
* **FSU100 Betting Engine** (`chimeracloud/fsu100`) — runs the same plugin
  against the live Betfair stream and (in `LIVE` mode) places real lay
  bets.

Both tools share the same evaluator and the same plugin schema. A plugin
authored here works in both without modification.

## What's in the May-2026 ruleset

```
rule_1     odds 1.01 – 2.00     stake 3.0 pts
rule_2a    odds 2.00 – 3.00     stake 0   pts   (skip — see note)
rule_2b    odds 3.00 – 4.00     stake 1   pts
rule_2c    odds 4.00 – 5.00     stake 2   pts
rule_3a    odds 5.00 – 1000     stake 1.0 pts   gap_lt   2.0   also_lay_2nd
rule_3b    odds 5.00 – 1000     stake 1.0 pts   gap_gte  2.0
```

Rule 2 is split into three sub-bands so the operator can tune the
favourite mid-range independently. Rule 3 splits by gap-to-2nd: tight
spread → dual lay (3a), wide spread → single lay (3b).

### Controls

* `jofs_enabled` / `jofs_spread` — Joint Odds Favourite Splitting
* `spread_control` — block tight markets unless JOFS is enabled
* `mark_floor_enabled` / `mark_floor` — operator-set hard price floor
* `mark_ceiling_enabled` / `mark_ceiling` — operator-set hard price ceiling
* `mark_uplift_enabled` / `mark_uplift` — stake multiplier
* `signal_*` — placeholder gates (overround, field size, steam, band perf)
* `top2_concentration_enabled` plus `top2_06/07/08_*` — three-tier TOP2
  suppressor (Medium/Strong/Block) tuned by overround, fav ceiling,
  gap-to-3rd, and an optional stake multiplier

All `_enabled` flags default to `false` so a fresh plugin runs the base
rules only. Toggle them on as Mark validates each control set.

## How to use

### Configurator (recommended)

The Chimera portal at `https://chimerasportstrading.com/strategy/configurator`
loads this JSON, renders every value as an editable input, and lets the
operator save / load / send-to-backtest / apply-to-live without touching
any code.

### Manual

```bash
cp plugins/chimera_may2026_v1.json /Users/charles/Projects/fsu100/plugins/
# edit if you need a tuned variant
gcloud run deploy fsu100 ...   # Charles deploys via the GCP Console
```

After deploy, switch the engine to the new plugin via:

```bash
curl -X PUT "$PORTAL_API/api/proxy/fsu100/admin/config" \
  -H "Authorization: Bearer $PORTAL_JWT" \
  -H 'Content-Type: application/json' \
  -d '{"active_plugin": "chimera_may2026_v1", ...}'
```

## Evaluator behaviour notes

The shared evaluator (`fsu-bt/evaluator.py`, identical in both tools) is
intentionally generic — it iterates `strategy.rules`, picks the first
whose `odds_band` contains the favourite price, and applies any
`gap_lt` / `gap_gte` constraint. Rule **names are labels only**;
`rule_2a` and `banana_2a` flow through the same code path.

Two semantics in this plugin **rely on a small evaluator update** that
hasn't shipped yet:

* `enabled: false` on a rule — should cause the rule to be skipped
  entirely. The configurator UI filters disabled rules out before sending
  to the engine, so plugins submitted via the portal are unaffected. A
  plugin loaded directly from the file system would currently still run
  disabled rules; track this as an evaluator follow-up.
* `base_stake: 0` — should also cause the rule to skip. Same workaround
  as above.

When the evaluator is taught to honour both, the configurator's pre-send
filter can be removed.

## Compatibility

* `backtest-tool` `>= 1.0.0`
* `fsu100` `>= 1.0.0`

Both tools are deployed at `cst-api/api/proxy/<service>` behind the
Chimera portal; nothing in this plugin assumes a specific deploy URL.
