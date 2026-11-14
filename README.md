# FAA Outage Attribution Forecaster — make.fun entry

Forecasts the Yes price of each of Polymarket's FAA outage attribution markets
(Iran / foreign actor / hack / AI sandbox escape) at every minute of the 4 hours
after a checkpoint, as five quantiles (0.05, 0.25, 0.5, 0.75, 0.95), scored by
mean pinball loss.

## How it works

`main.ts` (Node 22, standard library only) reads every `<id>.in` case file in
the input directory and writes `<id>.out` for each. For every market it
forecasts

```
q_tau(h) = clamp(x + z_tau * sigma * sqrt(h), 0, 1)   (6 decimals)
```

where `x` is the market's last price, `z_tau` is the standard normal's quantile
at each level, and `sigma` is estimated from **that market's own recent
history in the case**:

```
sigma = clip(0.75 * mean|1-minute move| over the last 360 minutes
             * sqrt(pi/2), 0.0005, 0.02)
```

The median stays at the last price (no drift): trend-chasing was tested on the
practice and always hurt, even heavily shrunk. Heavy-tailed (Student-t)
quantiles were tested too and did not help. The edge over the house baseline
(fixed sigma 0.0025) is adaptivity: most minutes are quiet, so a fixed spread
pays at every quiet minute, while a market that just moved deserves a wider
spread — and each market gets its own number, re-estimated per case.

## Local dev grade (practice cases, vs Polymarket's actual prices)

Scored locally against the public 1-minute price history (fetchable again from
the CLOB `prices-history` endpoint; the practice is the NVIDIA CEO event):

| entry | loss | median err | 90% coverage |
|---|---|---|---|
| this entry | **0.278830** | 0.767503 | 0.889844 |
| house baseline | 0.363589 | 0.767503 | 0.949642 |

−23.3% vs the baseline, beating it on every one of the six practice cases
(p00 0.450560 vs 0.536138, p01 0.319073 vs 0.380352, p02 0.364186 vs 0.408408,
p03 0.268828 vs 0.368599, p04 0.211746 vs 0.298794, p05 0.120112 vs 0.238232). The practice is a different event
from the final's FAA family, so its score is no guide to the final — but the
estimator only uses each case's own history, so it carries over to any market.
Parameters sit in a flat region of the optimum (lookback 240–720 min,
multiplier 0.6–1.0 all score 0.279–0.283), not on a tuned spike.

## Run locally

```sh
mkdir -p out && MAKEFUN_INPUT_DIR=input MAKEFUN_OUTPUT_DIR=out node main.ts
```

One case runs in ~0.17 s (cap is 8 s); two runs write byte-identical output.
