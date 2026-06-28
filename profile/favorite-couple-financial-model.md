# Favorite Couple — Financial Model

*Single-country MVP: one monthly competition. All figures in EUR per month unless stated. Founder salary is excluded from operating costs and noted separately, since early-stage founders often don't pay themselves.*

> **All numbers below are editable assumptions, not predictions.** Change the drivers in Section 2 and the rest recalculates from the formulas shown.

---

## 1. How the Model Works

Revenue is driven by two populations — **voters** and **couples** — plus **sponsors**. Costs are mostly semi-fixed at a given scale, with payment processing scaling on transaction volume. The prize is a cost the company sets per round, not a percentage of revenue.

The model runs three scenarios — **Conservative**, **Base**, **Optimistic** — so you can see the range, not a single guess.

---

## 2. Driver Assumptions

| Driver | Conservative | Base | Optimistic |
|---|---:|---:|---:|
| Registered voters | 5,000 | 10,000 | 25,000 |
| Paying-voter conversion | 2% | 3% | 5% |
| → Paying voters | 100 | 300 | 1,250 |
| Avg monthly spend / paying voter | €5 | €8 | €12 |
| Competing couples | 200 | 500 | 1,200 |
| Boost price (monthly level) | €5 | €5 | €5 |
| Boost uptake (of couples) | 10% | 15% | 20% |
| Premium subscription price | €5 | €5 | €5 |
| Premium uptake (of couples) | 8% | 10% | 15% |
| Sponsor logo/ad income | €0 | €500 | €1,500 |
| Display ad income | €50 | €100 | €400 |
| Prize (company-set, affordable) | €200 | €500 | €1,500 |

---

## 3. Revenue Model

**Formulas**

```
Paid votes      = Paying voters × Avg monthly spend
Boost revenue   = Couples × Boost uptake × Boost price
Premium revenue = Couples × Premium uptake × Premium price
Sponsor income  = (assumption)
Display ads      = (assumption)
Total revenue   = Paid votes + Boost + Premium + Sponsor + Display ads
```

**Computed**

| Revenue stream | Conservative | Base | Optimistic |
|---|---:|---:|---:|
| Paid votes | €500 | €2,400 | €15,000 |
| Boost revenue | €100 | €375 | €1,200 |
| Premium subscriptions | €80 | €250 | €900 |
| Sponsor logo/ads | €0 | €500 | €1,500 |
| Display ads | €50 | €100 | €400 |
| **Total revenue** | **€730** | **€3,625** | **€19,000** |

Paid votes dominate revenue in every scenario — they are the lever that matters most.

---

## 4. Cost Model

**Formula for processing**

```
Payment processing ≈ 3.5% × (Paid votes + Boost + Premium) + fixed buffer
```

All other lines are semi-fixed assumptions at the given scale.

| Cost line | Conservative | Base | Optimistic |
|---|---:|---:|---:|
| Prize | €200 | €500 | €1,500 |
| Payment processing | €40 | €120 | €560 |
| Hosting / infrastructure | €100 | €200 | €500 |
| Moderation & support | €300 | €500 | €1,200 |
| Fraud prevention tools | €100 | €150 | €400 |
| Marketing / customer acquisition | €500 | €1,500 | €5,000 |
| Legal & compliance (amortized) | €200 | €300 | €500 |
| Tools / admin / misc | €150 | €250 | €600 |
| **Total cost** | **€1,590** | **€3,520** | **€10,260** |

---

## 5. Monthly P&L Summary

| | Conservative | Base | Optimistic |
|---|---:|---:|---:|
| Total revenue | €730 | €3,625 | €19,000 |
| Total cost | €1,590 | €3,520 | €10,260 |
| **Net (excl. founder salary)** | **−€860** | **+€105** | **+€8,740** |

What this says: at conservative uptake the MVP loses money; at base assumptions it roughly breaks even before paying the founder; at optimistic uptake it is strongly profitable. **If you pay yourself a salary, the base case turns negative** — so reaching profitability means pushing past base on voter count and conversion.

---

## 6. Break-Even

Holding the base assumptions for couples, sponsors, and ads fixed, and solving for the paid-vote revenue needed to cover operating costs (excluding founder salary):

```
Break-even paid-vote revenue ≈ €2,300 / month
  ≈ 290 paying voters  (at €8 avg spend)
  ≈ 9,700 registered voters  (at 3% conversion)
```

So the single-country MVP needs on the order of **~9,700 registered voters** at base assumptions to cover its own running costs. Below that, it runs at a loss; above it, margin opens quickly because most costs are semi-fixed.

**Levers that lower break-even:**
- Higher conversion (better paid-vote UX, compelling reasons to buy) — most powerful.
- Higher average spend (bundle pricing, boost demand).
- Sponsor income (drops straight to the bottom line — no processing cost).
- Lower customer-acquisition cost (couples bringing their own audiences is effectively free acquisition).

---

## 7. Illustrative 12-Month Path (Base Case)

Assumes marketing spend (inside costs) plus couples bringing their audiences drives voter growth; sponsors come online from month 4; revenue scales at roughly €0.37–0.40 per registered voter per month, improving slightly as sponsors are added. **Illustrative — directional, not a forecast.**

| Month | Registered voters | Revenue | Cost | Net | Cumulative |
|---:|---:|---:|---:|---:|---:|
| 1 | 2,000 | €730 | €1,800 | −€1,070 | −€1,070 |
| 2 | 3,500 | €1,300 | €2,100 | −€800 | −€1,870 |
| 3 | 5,500 | €2,050 | €2,600 | −€550 | −€2,420 |
| 4 | 8,000 | €3,100 | €3,200 | −€100 | −€2,520 |
| 5 | 11,000 | €4,300 | €3,700 | +€600 | −€1,920 |
| 6 | 14,500 | €5,700 | €4,300 | +€1,400 | −€520 |
| 7 | 18,000 | €7,100 | €4,900 | +€2,200 | +€1,680 |
| 8 | 22,000 | €8,700 | €5,600 | +€3,100 | +€4,780 |
| 9 | 26,000 | €10,200 | €6,200 | +€4,000 | +€8,780 |
| 10 | 30,000 | €11,800 | €6,800 | +€5,000 | +€13,780 |
| 11 | 34,000 | €13,400 | €7,400 | +€6,000 | +€19,780 |
| 12 | 38,000 | €15,000 | €8,000 | +€7,000 | +€26,780 |

In this path the business turns **monthly profitable around month 5** and recovers cumulative losses (pays back its early burn) around **month 7**. Peak cumulative cash needed before turning positive is roughly **€2,500** — the total you'd need to fund the launch through the loss-making months. That number is small because customer acquisition leans on couples bringing their own followers; if paid marketing has to do the heavy lifting instead, both costs and the funding requirement rise substantially.

---

## 8. Sensitivity — Which Levers Move the Outcome Most

Ranked by impact on monthly net, strongest first:

1. **Registered voter count** — drives paid votes, the largest revenue line. Growth here is mostly powered by couples' own audiences, so couple acquisition is really voter acquisition.
2. **Paying-voter conversion %** — going from 3% to 4% at 10,000 voters adds ~€800/month in paid votes at base spend.
3. **Average spend per paying voter** — bundle design and boost demand. From €8 to €10 at 300 payers adds €600/month.
4. **Sponsor income** — pure margin, no processing cost, and can fund the prize entirely.
5. **Marketing efficiency** — the cost lever; if couples drive growth organically, this stays low and break-even drops sharply.

---

## 9. Key Assumptions & Caveats

- Founder salary excluded throughout; include it as a fixed cost line once you draw one.
- Payment processing modeled at ~3.5% + buffer; confirm with your actual provider.
- Conversion (2–5%) and average spend (€5–12) are the least certain inputs and should be revisited the moment you have real data — they move the whole model.
- The 12-month path assumes growth is driven largely by couples' own audiences (low-cost acquisition). If you rely on paid ads instead, costs and required funding rise.
- This is single-country MVP economics. Adding the regional/continental/global ladder multiplies both engagement and operational complexity (more competitions to run, moderate, and pay out) — model those separately when you get there.
- Prize is a cost the company sets and announces per round, funded from revenue and/or sponsors — not a percentage of incoming payments.

---

*Edit any driver in Section 2 to re-run. For live what-if testing across all these levers at once, this is best as a spreadsheet — say the word and I'll produce an editable .xlsx version.*
