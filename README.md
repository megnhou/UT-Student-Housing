# Austin Student Housing: Rent vs. Buy

A Python and JavaScript model that asks a practical question for UT Austin students and their families: **is it cheaper to buy a West Campus 2-bed, 2-bath condo, live in one room, and rent out the other, or to rent a bedroom and invest the money instead?**

[Live interactive version](ADD-YOUR-LINK-HERE) · Data gathered October 2026

> This is a learning project, not financial advice.

---

## The question

Buying looks attractive on paper. A condo near campus can pay part of its own way through roommate rent, and the owner builds equity instead of paying a landlord. But buying also ties up a lot of cash, adds fees, and carries price risk. The goal here is to compare the two paths fairly and find out *what has to be true* for buying to win.

## How the model works

Both paths start with the same cash and spend the **same total cash each month**. Whichever path has the lower monthly outlay invests the difference, so neither side gets a free advantage.

- **Buy path:** pay the down payment and closing costs, then monthly mortgage, property tax, HOA, insurance, maintenance, and utilities. Subtract roommate rent (after vacancy) and any utilities the roommate covers. Sell at the end, paying selling costs and any remaining loan.
- **Rent path:** rent one bedroom, pay a share of utilities, parking, and renters insurance. Invest the cash that would have gone into the condo, plus any monthly savings.
- **Result:** net worth for each path, year by year, and the difference (`buy_advantage`). A positive number means buying wins.

This framing is what makes the comparison honest: the real cost of owning includes the **opportunity cost** of the cash tied up in the condo, not just the bills.

## Data

Inputs come from active listings and public sources (searched October 2026), with anything I couldn't find clearly labeled as an assumption.

| Input | Default | Source / note |
|---|---|---|
| Condo price | $360,000 | West Campus 2x2 listings, about $250K to $420K |
| HOA fee | $380/month | Listings range $285 to $495 |
| Property tax | 2.0% of value | Listing tax bills imply 1.8% to 2.2%; no homestead exemption since the owner doesn't live there |
| Roommate rent | $1,125/month | Whole 2x2 units rent for about $2,100 to $2,400 |
| Mortgage rate | 7.75% | Freddie Mac 30-year average (7.28% on Oct 1, 2026) plus about 0.5 points for non-owner-occupied loans (*assumption*) |
| Utilities | $200/month, whole unit | Austin Energy typical bill and internet costs |
| Insurance, maintenance | $1,100/year, 0.5% of value | *Assumption* |
| Vacancy | 8% | *Assumption* |
| Parking if renting | $75/month | *Assumption*; most condo listings include two spaces |
| Investment return | 5% a year | *Assumption*; the cost of tying up cash |
| Condo price growth | 1% a year | *Assumption*; Austin condo medians were falling in early 2026 |

The data is a **small sample of listings** (asking prices, not sale prices), not a full market survey.

## Key findings (default scenario: $360K condo, 25% down)

- Owning costs the family about **$2,220 a month** net after the roommate pays, compared with about **$1,320** to rent a bedroom.
- After 4 years, renting and investing leaves the family about **$76K ahead**, even after counting the roommate income.
- Buying a **smaller loan helps but doesn't flip the result.** Paying all cash cuts the gap to about $42K after 4 years.
- Buying wins only when several things line up. For example: all cash, a low return on the alternative (3%), and condo prices rising about 3% a year gives buying a lead of about **$16K** after 4 years.
- **Condo price growth** drives the outcome the most:

| Condo price change | 2 years | 4 years | 6 years | 8 years |
|---|---|---|---|---|
| −5% per year | −$92K | −$148K | −$200K | −$248K |
| −2% per year | −$73K | −$114K | −$154K | −$193K |
| 0% per year | −$60K | −$89K | −$119K | −$149K |
| +2% per year | −$47K | −$63K | −$79K | −$98K |
| +5% per year | −$26K | −$20K | −$13K | −$5K |

*Buy advantage vs. renting. Negative means renting wins.*

**Why:** after HOA, property tax, insurance, and maintenance, the unit earns a thin return (roughly 3% before financing), which is below what the same cash could earn elsewhere. Borrowing at about 7.75% makes that worse.

## Run it

```bash
pip install pandas matplotlib   # matplotlib is optional
python west_campus_2x2.py
```

To test your own scenario:

```python
from dataclasses import replace
from west_campus_2x2 import Scenario, simulate, breakeven_year

s = Scenario(price=300_000, down_pct=1.0, hoa_monthly=300, roommate_rent=1_000)
df = simulate(s)
print(df[["year", "buy_net_worth", "rent_net_worth", "buy_advantage"]])
```

The interactive page (`west-campus-2x2.html`) is a single file that runs in any browser. It has sliders for condo price change, down payment, mortgage rate, years held, and investment return.

## Validation

The model is implemented twice, in Python and in JavaScript. I checked that both versions give identical results (to the dollar) on several scenarios before publishing.

## Limitations

- Small listing sample and asking prices, not a statistical market study.
- Ignores income tax on rental income (partly offset by depreciation and expenses), capital gains tax on sale, and Texas property tax appraisal caps.
- Assumes steady returns and steady price growth. Real returns and prices vary a lot, so a Monte Carlo version would show the range.
- Doesn't model the time and legal responsibility of being a landlord or HOA rules on renting.
- A cash buyer's real alternative may not be a diversified investment portfolio, and the right return to use is a judgment call. The page lets you change it.

## Project files

| File | What it is |
|---|---|
| `west_campus_2x2.py` | Python model and sensitivity analysis |
| `west-campus-2x2.html` | Interactive tool (single file) |
| `rent_vs_buy.py` | General Austin condo vs. apartment model |
| `austin-rent-vs-buy.html` | Interactive tool for the general model |

## Ideas for next steps

- Monte Carlo simulation of price and return paths
- A "keep or sell at graduation" comparison
- Pull listing data from a public dataset instead of by hand
- Add federal income tax on rental income

*Built with help from Claude (Anthropic).*
