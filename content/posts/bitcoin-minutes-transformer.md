---
date: 2026-01-31
title: "When a trading metric measures the base rate"
description: "A nanochat-style GPT on 7.4 million Bitcoin minutes. Val directional accuracy 56.6%. Zero trades. A Sharpe of 23.9 that is always-down annualized at 525,600 periods per year."
tags:
  - machine learning
  - transformers
  - evaluation
  - Bitcoin
---

A language model trained on prices is usually sold as a trading system. The more honest description is a classification problem with a noisy label and a metric that can be rewritten until it looks like a fund. This post is about that rewrite.

The repo started as a byte-level GPT (`PLAN.md`: vocabulary 256, no tokenizer, context 256, overfit one batch until the loss collapses). The architecture that ended up in `model.py` left the Phase B sketch behind for the small-transformer recipe from [Karpathy’s nanochat](https://github.com/karpathy/nanochat): RoPE (Su et al., RoFormer), QK-norm, ReLU² (Primer, via nanochat), RMSNorm, separate Q/K/V, no bias, untied embeddings. None of that is mine. `NANOCHAT_ADAPTATIONS.md` says so, and so do the comments in the attention block.

This post focuses on the retarget onto Bitcoin 1-minute log-returns, and on an evaluation file that contains both `sharpe_ratio: 0.0` and `strategy_sharpe: 23.91`. Related work on Stockformer, temporal fusion transformers, and “nanochat is 4× faster” brochure numbers is out of scope. The adaptation notes list expected Flash Attention and BF16 gains. I am not citing those as measurements. Both Bitcoin runs below logged `train/dtype: float32` on CPU of an Apple M3 MacBook Air.

I explicitly mention the two Sharpes because they deserve separate names: one looks at trades, the other at every bar.

## Minutes, not bytes

The Kaggle 1-minute BTC series (`mczielinski/bitcoin-historical-data`) becomes discrete tokens the same way bytes did, except the alphabet is a quantized return. `prepare_data_bitcoin.py` takes log-returns, clips them to mean ± 3σ, and min–max scales into 256 bins. The prediction target is the next bin; dollars stay out of it. Absolute price from $4.58 (first close in `metadata.json`) to 2020s Bitcoin is a non-stationary mess. A 256-way softmax over clipped returns is at least a stationary-ish classification problem.

Retail indicators are tokenized the same way:

| Feature | Bins |
|---|---:|
| price log-return | 256 |
| RSI(14) | 64 |
| ATR(14), percentile | 64 |
| Bollinger position (20, 2σ) | 32 |
| relative volume (20) | 32 |
| direction (down / flat / up) | 3 |

`data/metadata.json` records **7,404,159** rows after the return diff. On disk: `train_features.npy` `(6,663,743, 6)` and `val_features.npy` `(740,416, 6)`, a 90/10 chronological split. Return bounds after clipping are about ±0.532% per bar.

`MultiFeatureEmbedding` gives every stream its own table and, in default `sum` mode, a softmax over six learned weights. A 3-way direction head on the same hidden state is optional, for a multi-task loss.

```python
self.embeddings = nn.ModuleDict({
    name: nn.Embedding(vocab_size, n_embed)
    for name, vocab_size in feature_vocab_sizes.items()
})
self.feature_weights = nn.Parameter(torch.ones(self.n_features))
self.direction_head = nn.Linear(n_embed, 3, bias=False)  # down, flat, up
```

That is the whole trick on the data side: keep the transformer, change what a token means, add features a human already looks at. The result is less a new architecture than a discretization.

## Two laptop runs

Everything below quotes wandb summaries; no cleaned leaderboard sits behind them.

**Multi-feature, 680,710 parameters** — wandb `yqiaqx1w`, `--use-direction-head`, Apple M3, 16 GB, **CPU**, float32. Default trainer shape: `n_embed=128`, `n_layer=3`, `n_head=4`.

| Metric | Value |
|---|---:|
| `model/total_params` | 680,710 |
| `train/loss` | 4.5178 |
| `val/loss` | 4.9784 |
| `train/dir_accuracy` | 0.6502 |
| `val/dir_accuracy` | 0.5664 |
| `val/weighted_dir_accuracy` | 0.5024 |
| `_runtime` | 313 s |

`output.log`:

```
Step      0 | train_loss: 5.8747 | val_loss: 5.8747 | dir_acc: 0.518
Step    100 | train_loss: 5.1097 | val_loss: 4.9784 | dir_acc: 0.566
KeyboardInterrupt
```

The val numbers are the step-100 eval. Training continued to about step 179 and I hit Ctrl-C. A 256-class cross-entropy starts near `ln(256) ≈ 5.55`; 5.87 at step 0 is a cold start, 4.98 val at step 100 is a real drop, and it is still a **short** run.

**Single-feature, 3,276,800 parameters** — wandb `owdh14sw`. Same M3, CPU, float32. `n_embed=256`, `n_layer=4`. Resumed from `checkpoints/best_model.pt` at step 7500. Last logged: `train/loss` 3.5386, `val/loss` 3.7919. Then another KeyboardInterrupt. No directional-accuracy key on that summary, so I will not invent one.

The larger model has the lower cross-entropy. That is what CE is for, and it says little on its own about when to send a market order.

There is a pytest suite (`TEST_SUMMARY.md` lists 101 tests: shapes, causal attention, RoPE, multi-feature embeddings, a training step). Insurance, basically.

## Pattern: two metrics, one JSON

`results.json` is `backtest.py` on the val split, threshold 5 bins, 1 bp of transaction cost, not long-only:

```json
{
  "total_return": 0.0,
  "sharpe_ratio": 0.0,
  "num_trades": 0,
  "metrics": {
    "directional_accuracy": 0.5661202525403326,
    "weighted_directional_accuracy": 0.5009059606132406,
    "up_accuracy": 0.0,
    "down_accuracy": 1.0,
    "strategy_sharpe": 23.91144414846091,
    "strategy_total_return": 14.990909536419654
  }
}
```

Read the top half first. **Zero trades.** The simulator that opens and closes positions did nothing, so it correctly reports return 0 and Sharpe 0. That is the trade backtest.

The nested `metrics` block is a different function, `compute_trading_metrics`. It never looks at the trade list.

```python
pred_up = predictions > middle_bin          # 127
actual_up = actuals > middle_bin
strategy_returns = np.where(pred_up, actual_returns, -actual_returns)
# ...
def compute_sharpe_ratio(returns, periods_per_year: int = 525600) -> float:
    annualized_return = mean_return * periods_per_year
    annualized_vol = std_return * np.sqrt(periods_per_year)
    return annualized_return / annualized_vol
```

Every bar is a position. Predicted token above 127 → take the next return. Otherwise flip the sign. No threshold, no cost, no holding period. `525600` is `365 × 24 × 60`. One-minute bars, every minute of a 365-day year, costless. A microscopic mean per minute, times `sqrt(525600) ≈ 725`, is how you manufacture a Sharpe of 23.9 out of a constant short.

`up_accuracy: 0.0` and `down_accuracy: 1.0` pin down what the constant was. Every predicted token was `≤ 127`. `pred_up` is always false. `strategy_returns` is always `-actual_returns`. The nested Sharpe is the annualized Sharpe of **always shorting the de-tokenized val series**. `strategy_total_return` sums clipped bin midpoints rather than compounding equity.

The 56.6% directional accuracy is the same fact from the other side. If you never predict up, your “directional accuracy” is the fraction of bars that are not up. The val window is a bit more down/flat than up, so a majority-class classifier scores 0.566. Wandb’s `val/dir_accuracy 0.5664` and `results.json`’s `0.56612` agree to three decimals. Weighted directional accuracy sits on 0.50 — once you weight by how far the true bin is from 127, the “edge” is gone.

Why zero trades *and* always-down? The live strategy is a dead zone around the middle bin:

```python
signals[predicted_tokens > self.middle_bin + self.threshold_bins] = 1   # > 132
signals[predicted_tokens < self.middle_bin - self.threshold_bins] = -1  # < 122
```

Always-down predictions that stay in `[122, 127]` count as down for the metric and as **flat** for the book. The book stays empty. `run_backtest` hits `len(trades) == 0` and returns Sharpe 0. That is the only reading that fits the JSON.

I left the 23.9 in the file. I am not putting it in a results table.

## What 56.6% is allowed to mean

A 680k transformer, six tokenized features, a direction head, 7.4 million minutes, and the honest sentence is: **on a 100-step laptop eval, next-minute direction on this val slice is the down base rate.**

Train directional accuracy 0.6502 against val 0.5664 is already a gap, and the run was still in warmup (`train/lr` 2.7e-4 toward 3e-4). Weighted val accuracy 0.5024 says the model is not better on the minutes that move.

None of this surprises me as a markets result. One-minute BTC log-returns after a 3σ clip are close to a small-noise classification problem. A causal transformer can memorize recent bin identities. It can also notice that “down or flat” is slightly more common than “up” and park its mass on the cheap side of bin 127. That is a legitimate minimum of the loss — anything but alpha.

I would not deploy this. I would not paper-trade it. `BITCOIN_README.md` already says the setup is educational. The part I care about is that the evaluation almost disagreed with that sentence, and the disagreement was a bug in the metric, not a hidden edge.

## Implications

If I picked this up again, the next instruments are boring on purpose.

First, a **base-rate table**: fraction of val bars above/below 127, and the accuracy of always-down, always-up, and last-bar copy. 56.6% has to beat those numbers rather than 50%.

Second, **separate the two Sharpes**. The trade simulator already does the right thing when it does not trade. The nested `strategy_sharpe` should not share a name with it. Costless sign-flips on 1-minute bars should not be annualized with 525,600 periods and then printed next to `num_trades`.

Third, a longer train than 179 steps, on a GPU, with an explicit “does it ever leave `[122, 127]`?” histogram. If the argmax never exits the dead zone, the strategy is defined to be flat.

Fourth, I would stop calling SDPA and BF16 a speedup until I measure them. This machine ran float32 on CPU at ~10k tokens/s in the 680k log — laptop territory, plain and simple.

The useful residue is the hygiene. A from-scratch training loop is a good way to learn why RoPE sits inside attention. A 7.4-million-row dataset is a good way to learn that more rows will not save a target that is mostly noise. A `results.json` that contains both `sharpe_ratio: 0.0` and `strategy_sharpe: 23.91` is a good way to learn which field you are allowed to read out loud.

I trained a transformer on 7.4 million Bitcoin minutes. It learned the base rate, parked on down, and never crossed the trade threshold. The impressive Sharpe is the annualization of that shrug. I am keeping the 56.6 percent. I am throwing the 23.9 away.
