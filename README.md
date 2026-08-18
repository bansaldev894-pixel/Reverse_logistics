# Where Is the Money Leaking? — A Reverse Logistics Analytics Project

Every e-commerce operation loses money to returns and RTOs (Return to Origin). Most teams track this with a single "return rate" number and stop there. This project exists because that single number hides more than it reveals — it can't tell you *which* category is actually the problem, whether common assumptions like "COD customers return more" are even true, or what a proposed fix is actually worth in rupees.

This repo is my attempt to answer those questions properly: a first-principles cost model, a full data-cleaning pipeline, and a two-page decision-support dashboard — built to answer 5 specific business questions a Head of Reverse Logistics would actually ask, and nothing more.

---

## Why This Project Exists

I set myself one rule before opening any tool: **every chart has to map back to a real business question, or it doesn't get built.** Dashboards fail not because they lack charts, but because they have too many that don't answer anything specific. So the whole project is scoped around 5 questions:

1. Which category is actually driving the loss — and is it a *frequency* problem or a *severity* problem?
2. Is the common belief that "COD customers return more" actually true?
3. What's the single headline number, unambiguous, first thing anyone should see?
4. Is this getting worse over time?
5. If we fix something, what does it actually save — and can someone test that themselves?

Everything below is organized around finding those answers.

---

## What's Actually in This Repo

| File | What it is |
|---|---|
| `REVERSE_LOGISTICS_DASHBOARD.pbix` | The two-page Power BI dashboard — diagnosis + decision support |
| `reverse_logistics_cost_model.xlsx` | The Excel cost engine that calculates loss for every event and models the same scenarios |
| `sql/` | The MySQL cleaning pipeline — raw tables → cleaning views → the final loss-attribution view |
| `data/` | CSV exports of the cleaned data, used to build the Power BI model |
| `images/` | Dashboard screenshots (below) |
| `LICENSE` | MIT |

---

## Step One: Building a Cost Model I Could Actually Trust

Before any dashboard, I had to answer a boring but critical question: **what does a "return" actually cost?** Not just the refund — someone pays for reverse shipping, someone inspects the item, someone restocks it (or writes it off). So every single one of 2,969 return/RTO events gets its own calculated loss:

```
loss = refund_amount + reverse_shipping_cost + qc_inspection_cost + restocking_cost − estimated_recovery
```

The one deliberate design choice I'm proudest of: **RTO and Customer Return are treated differently.** An RTO event (the order never reached the customer, usually a failed COD delivery) gets zero recovery credit — there's no refund to net against, the loss is purely wasted shipping and handling. A Customer Return (the item was delivered, then sent back) gets a recovery credit, because a returned item can usually be resold at a discount. Two events that both get labeled "return" in most reporting are, economically, completely different — and the model reflects that.

I built this three times, on purpose — as SQL views, as Excel formulas, and as Power BI DAX — and cross-checked all three against each other until they agreed. All three land on the same number: **₹15,35,803.60** in total annual loss across 2,969 events. That agreement is what let me trust the number enough to build a dashboard on top of it.

Along the way, cleaning the raw data surfaced real problems I had to fix before any of this math would be correct:
- **Duplicate customer records** in the raw CRM export, fixed by keeping only the most recent record per customer.
- **Inconsistent payment method labels** (multiple spellings/cases meaning "COD" or "Prepaid"), standardized into two clean categories.
- **A city name typo** ("bengaluru" vs "bangalore") that would have silently split one city into two in any geography analysis.

None of that is glamorous, but it's the difference between a number you can defend and a number that just looks right.

---

## Step Two: Diagnosing Where the Money Actually Goes

![Page 1 — Diagnostic Dashboard](images/page1-diagnostic.png)

This page exists to answer: **where is the money leaking, and why?**

**Finding 1 — Furniture and Fashion are not the same kind of problem.**
Furniture accounts for **₹0.75M of the ₹1.54M total** — by far the biggest single category — despite a comparatively low return rate. It's a *severity* problem: fewer events, each one expensive, driven by heavy shipping and restocking costs. Fashion is the mirror image: the highest return rate of any category, but a low average loss per event — a *volume* problem. You cannot fix these the same way. Furniture needs cheaper handling per event; Fashion needs fewer events happening at scale in the first place. The scatter chart makes this argument without a word of explanation — the two categories sit in opposite corners of the chart.

**Finding 2 — I tested the "COD returns more" assumption, and it's wrong in an interesting way.**
COD's RTO rate is **12.83%** versus Prepaid's **2.57%** — a real, large gap. But Customer Return rate — items that were actually delivered and then sent back — is nearly identical: **11.41% for COD vs. 11.93% for Prepaid.** COD customers don't change their minds more often once they receive a product. COD's problem is entirely in *getting the product delivered in the first place* — bad addresses, unreachable customers, payment not ready at the door. That reframes the fix completely: this isn't a "make COD customers commit more" project, it's a delivery-execution project.

**Finding 3 — I checked if it was getting worse, and stayed honest about what the data could and couldn't tell me.**
The monthly trend is flat across the year. Rather than spin that into "returns are stable," I put a caveat directly on the dashboard: this is a single year of synthetic data with no seasonal or festival demand modeled, so flat shouldn't be read as proof of real-world stability. Knowing what a finding *can't* claim is part of the finding.

---

## Step Three: Turning the Diagnosis Into a Decision

![Page 2 — Decision Support Dashboard](images/page2-decision-support.png)

Anyone can report what happened. The harder, more useful thing is building something that helps someone decide what to do next — so Page 2 answers a completely different question: **if we fix this, what do we actually get?**

Two live sliders sit at the top — one for reducing COD RTO events, one for reducing Fashion Customer Returns — each running 0–30%. Drag either one, and three cards update in real time: projected savings from each fix individually, and the combined total. At the levels shown above (20% COD RTO reduction, 15% Fashion reduction), that's **₹47,310 + ₹33,839 = ₹81,150** in projected annual savings — a number a manager can actually act on, not just admire.

One deliberate scoping decision worth explaining: the Fashion lever only targets **Customer Returns**, not RTO. A sizing/listing-accuracy fix only prevents someone from returning an item *after* they've received it — it has no effect on delivery failures. Scoping it that way also means the two savings levers never double-count the same event, so the combined number is honest, not inflated.

**Finding 4 — there's no single dominant cause, and that's itself a finding.**
Breaking down the top reasons behind COD RTO and Fashion returns separately, no single reason runs away with a majority in either case — "Payment Not Ready" (228 events) barely leads "Incorrect Address" (211) and "Customer Not Reachable" (200) for COD; "Changed Mind" (100) barely leads "Better Price Elsewhere" (96) for Fashion. That flatness has a real operational implication: don't chase one silver-bullet fix. Fund a portfolio of small, unglamorous improvements instead — because no single intervention will move the needle alone.

---

## What I Deliberately Left Out

I want to be upfront about this, because it's a discipline choice, not a limitation. No customer-level drill-down, no geography map, no product-level table — I had a rule that every visual had to map to one of the 5 business questions above, and stuck to it even when it would've been easy to add "one more chart." A Head of Reverse Logistics has five minutes, not fifty. Two focused pages that each answer a real question beat fifteen charts that answer none clearly.

---

## Assumptions & Limitations

- All cost figures (shipping, QC, restocking, markdown %, write-off %) are portfolio-project assumptions informed by general e-commerce reverse-logistics patterns — **not real company data.**
- Intervention reduction percentages on the sliders are illustrative assumptions to demonstrate the mechanism, not measured outcomes from a real fix.
- The dataset covers a single synthetic year with no seasonal/festival demand modeling.
- Blended Return Rate (18.63%) is calculated on eligible shipped items only, excluding Cancelled orders, which were never actually at risk of returning.

---

## The Short Version

*I built a cost model that calculates the true loss of every return and RTO event from first principles — not just the refund, but reverse shipping, inspection, restocking, and resale recovery — validated it three separate ways (SQL, Excel, DAX) until all three agreed, then built a two-page dashboard on top of it: one page diagnosing exactly where ₹15.4L in annual losses comes from and disproving a common industry assumption about COD, and a second page that turns that diagnosis into a live decision tool a manager can actually test before committing budget to a fix.*

---

**Author:** Dev Bansal · Business Analyst / Supply Chain Analytics
