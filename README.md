# technicalindicators-takt

[technicalindicators](https://www.npmjs.com/package/technicalindicators) 3.1.0 with a faster native core
for ten popular indicators. The API is the original's, unchanged. Every result is identical to the
original's, bit for bit.

Covered: **SMA, EMA, WMA, RSI, BollingerBands, MACD, ATR, ADX, CCI, Stochastic** (plus WEMA and SD). All
other functions of technicalindicators run the original code.

## Install

Keep your imports: install it under the original name with an npm alias.

```sh
npm install technicalindicators@npm:technicalindicators-takt
```

```js
const { RSI, BollingerBands } = require('technicalindicators');   // or: import { RSI } from 'technicalindicators'
RSI.calculate({ period: 14, values: closes });
```

Or install it under its own name and import from `technicalindicators-takt`.

## How much faster

All runs were on an AMD Ryzen 9 7950X3D (Zen 4) with Node 20, measured by
[takt-harness](https://github.com/abx-takt/takt-harness). Each figure is the median of 7 interleaved
rounds, in CPU cycles of the whole Node process, with a 95% bootstrap confidence interval. The output
hash was identical in every run.

**Parameter sweep: 20 parameter values over 200,000 bars**

| Indicator | Factor (95% CI) | Wall time, original → takt |
|---|---:|---|
| CCI | **23.0×** (22.6–23.4) | 5.46 s → 0.30 s |
| BollingerBands | **11.7×** (11.4–11.9) | 6.99 s → 0.80 s |
| WMA | **6.8×** (6.7–6.9) | 2.26 s → 0.35 s |
| RSI | **6.2×** (6.1–6.3) | 0.83 s → 0.18 s |
| SMA | **5.1×** (4.9–5.2) | 0.69 s → 0.15 s |
| Stochastic | **4.5×** (4.5–4.7) | 2.79 s → 0.69 s |
| ATR | **3.8×** (3.8–3.8) | 0.49 s → 0.18 s |
| EMA | 1.9× (1.9–2.0) | 0.25 s → 0.15 s |
| MACD | 1.7× (1.6–1.7) | 1.19 s → 0.79 s |
| ADX | 1.7× (1.6–2.6), 15 rounds | 1.76 s → 1.16 s |

**One call on 1,000,000 bars, default parameters**

| Indicator | The `calculate()` call itself, original → takt | Whole process, factor (95% CI) |
|---|---|---:|
| CCI | 714 ms → 44 ms (16×) | 7.0× (6.9–7.0) |
| SMA | 205 ms → 22 ms (9.2×) | 2.5× (2.4–2.6) |
| WMA | 333 ms → 42 ms (8.0×) | 3.3× (3.3–3.3) |
| BollingerBands | 943 ms → 135 ms (7.0×) | 5.0× (5.0–5.1) |
| Stochastic | 568 ms → 106 ms (5.4×) | 3.3× (3.1–3.5) |
| RSI | 190 ms → 38 ms (5.0×) | 2.5× (2.5–2.6) |
| ADX | 396 ms → 125 ms (3.2×) | 2.8× (2.8–2.9) |
| ATR | 97 ms → 33 ms (3.0×) | 1.7× (1.6–1.7) |
| EMA | 47 ms → 21 ms (2.2×) | 1.2× (1.1–1.2) |
| MACD | 266 ms → 126 ms (2.1×) | 1.7× (1.7–1.8) |

The "call itself" column is the median of 3 runs of `calculate()`, timed inside the process. The
whole-process factor also includes starting Node and generating the 1,000,000 bars.

EMA is cheap in the original too: a warm call on 1,000,000 bars takes 39 ms there and 8.9 ms here. Of
those 8.9 ms, 7 go to reading the JavaScript input array and building the result array, and the native
EMA itself takes 0.34 ms. EMA therefore gains 4.4× per warm call and 6.4× over 99 periods on 1,000,000
bars (SMA: 22×), but less in the short whole-process runs above.

MACD, ADX and Stochastic return one object per bar. Both packages spend most of their time
allocating those objects and collecting garbage, so the gain is smaller there. For the same reason,
ADX sweeps vary from run to run.

To reproduce: `node bench/run.js <orig|takt> <sweep|single> <Indicator>`. The readings are in
`results/`.

## The same output, bit for bit

`verify/compare.js` compares every covered indicator with the original technicalindicators 3.1.0 that
this package installs. It covers every API form:
- `X.calculate()` and `x()`;
- `new X().result` and `getResult()`;
- `nextValue()` after construction.

The inputs include random walks, cent-rounded and integer prices, zeros, negative values, NaN, ±Infinity,
−0, magnitudes of 1e±300, constant and alternating series, high below low, and lengths from 0 to 2,000,
with periods 1–26 and several MACD, Bollinger and Stochastic settings. Numbers are compared exactly
(`Object.is`), objects key by key.

Results:
- **Full set** (lengths 0–2,000, periods 1–26, 708 series): 600,391 comparisons, 0 differences, on
  Node 18 (AMD Zen 3).
- **Quick set** (`--quick`: lengths 0–300, periods 1–14, 336 series): 220,423 comparisons, 0 differences,
  on Node 18 (Zen 3) and on Node 20 (Zen 4).

Run the check yourself:

```sh
node node_modules/technicalindicators/verify/compare.js --quick
```

On your own data, run your program twice and compare the outputs. `TAKT_TI_DISABLE=1` makes this package
run the original code everywhere:

```sh
TAKT_TI_DISABLE=1 node your-backtest.js > original.txt
node your-backtest.js > takt.txt
cmp original.txt takt.txt
```

## Limits

- **Platforms.** The native core is prebuilt for Linux x86-64 with a CPU from about 2009 or newer
  (SSE4.2). On other platforms, older CPUs and in browsers, the package runs the original code: same
  results, original speed.
- **Inputs.** The fast path takes plain inputs: arrays of numbers and integer periods. Inputs with
  `reversedInput`, a custom `format`, a global `setConfig('precision', …)`, holes, non-number elements,
  fractional periods, or high/low/close arrays of different lengths go to the original code unchanged.
- **Streaming.** `nextValue()` after construction gives the original's results by replaying the
  series through the original code once. Streaming is no faster than the original.

## Your own code, this fast?

TAKT speeds up the hot code of backtests, parameter sweeps and model training, with output identical
to the original, and measures the speedup before you pay. Free speed check:
https://taktcycles.com/?src=github · Measurements and updates: https://t.me/taktcycles

## Licence

- technicalindicators is MIT-licensed (© 2016 Anand Aravindan) and installed unchanged as a dependency.
- The JavaScript layer of this package is MIT.
- The native core is free to use and to redistribute with this package; its source is not published.

See `LICENSE` and `NOTICE`. This package is not affiliated with the technicalindicators project.

Questions, bugs, an indicator you want faster: https://github.com/abx-takt/technicalindicators-takt/issues
