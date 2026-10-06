# UT-Student-Housing
monthly cash-flow simulation comparing buying a West Campus 2x2 condo and renting out a room against renting a bedroom and investing the same cash
"""
West Campus 2x2: buy a condo and rent out the spare room, vs. rent a bedroom.

Two paths, same starting cash:
  BUY   Family buys a 2-bed/2-bath condo near UT. The student lives in one room, a
        roommate rents the other. The family pays all ownership costs and sells later.
  RENT  The student rents one bedroom in a similar 2x2 and invests the cash that
        would have gone into the condo.

Each month, whichever path has the lower cash outlay invests the difference, so both
families spend identical cash. We compare net worth at the end.

Data used for defaults (searched Oct 2026):
  * West Campus 2x2 condos listed for sale: about $250K-$420K, most near $360K.
  * HOA fees on those listings: $285-$495/month (often cover water, trash, parking).
  * Condos include 2 reserved parking spaces in most listings.
  * Whole 2x2 units rent for about $2,100-$2,400/month (about $1,050-$1,200 per room).
  * Annual tax bills on listings: $6,500-$8,128 (1.8%-2.2% of price; no homestead
    exemption, because the owner does not live there).
  * Utilities (Austin Energy average bill ~$117-$150, internet $50-$80, water/trash
    often in HOA or a ~$65 flat fee).
Assumptions I could not find solid data for (marked ASSUMPTION): insurance, maintenance,
vacancy, rental parking cost, investment-loan rate premium.
"""
from dataclasses import dataclass, replace
import pandas as pd


@dataclass
class Scenario:
    # Buying the condo
    price: float = 360_000
    down_pct: float = 0.25            # 1.0 = all cash. Non-owner-occupied loans usually need 15-25%
    mortgage_rate: float = 0.0775     # ASSUMPTION: Freddie Mac ~7.28% + ~0.5% investment/second-home premium
    term_years: int = 30
    property_tax_rate: float = 0.020  # listings imply 1.8%-2.2%; no homestead exemption
    hoa_monthly: float = 380          # listings: $285-$495
    insurance_annual: float = 1_100   # ASSUMPTION: landlord/condo policy
    maintenance_pct: float = 0.005    # ASSUMPTION, of value per year
    pmi_rate: float = 0.005           # annual, of loan, if balance > 80% of price
    closing_pct: float = 0.03
    selling_pct: float = 0.06
    appreciation: float = 0.01        # recent Austin condo medians have been falling
    # Roommate side
    roommate_rent: float = 1_125      # half of a ~$2,250 whole-unit rent
    vacancy: float = 0.08             # ASSUMPTION: about one month empty per year
    utilities_total: float = 200      # electric ~$140 + internet ~$60; water/trash in HOA
    roommate_util_share: float = 0.5
    # If renting instead
    own_bedroom_rent: float = 1_125
    renter_utilities: float = 100     # your half
    renter_parking: float = 75        # ASSUMPTION: many complexes include it, others charge $100+
    renters_insurance_annual: float = 180
    # Growth and investing
    rent_growth: float = 0.02
    cost_growth: float = 0.03         # HOA, insurance, utilities, parking
    invest_return: float = 0.05
    years: int = 10


def monthly_payment(loan, annual_rate, years):
    n, r = years * 12, annual_rate / 12
    return loan / n if r == 0 else loan * r / (1 - (1 + r) ** -n)


def simulate(s: Scenario) -> pd.DataFrame:
    loan = s.price * (1 - s.down_pct)
    pmt = monthly_payment(loan, s.mortgage_rate, s.term_years) if loan > 0 else 0.0
    rm = s.mortgage_rate / 12
    g_inv = (1 + s.invest_return) ** (1 / 12) - 1
    g_home = (1 + s.appreciation) ** (1 / 12) - 1

    balance, value = loan, s.price
    hoa, ins, util = s.hoa_monthly, s.insurance_annual / 12, s.utilities_total
    room, mine = s.roommate_rent, s.own_bedroom_rent
    r_util, r_park, r_ins = s.renter_utilities, s.renter_parking, s.renters_insurance_annual / 12
    renter_port = s.price * s.down_pct + s.price * s.closing_pct
    owner_port = 0.0
    rows = []

    for m in range(1, s.years * 12 + 1):
        interest = balance * rm
        principal = min(pmt - interest, balance) if balance > 0 else 0.0
        balance -= principal
        paying = pmt if (balance > 0 or principal > 0) else 0.0
        pmi = loan * s.pmi_rate / 12 if balance > 0.8 * s.price else 0.0
        tax = value * s.property_tax_rate / 12
        maint = value * s.maintenance_pct / 12

        owner_out = paying + tax + hoa + ins + maint + pmi + util
        owner_in = room * (1 - s.vacancy) + util * s.roommate_util_share
        owner_net = owner_out - owner_in
        renter_out = mine + r_util + r_park + r_ins
        diff = owner_net - renter_out

        renter_port *= 1 + g_inv
        owner_port *= 1 + g_inv
        if diff > 0:
            renter_port += diff
        else:
            owner_port += -diff

        value *= 1 + g_home
        if m % 12 == 0:
            o = value * (1 - s.selling_pct) - balance + owner_port
            rows.append({"year": m // 12, "home_value": value, "loan_balance": balance,
                         "buy_net_worth": o, "rent_net_worth": renter_port,
                         "buy_advantage": o - renter_port})
            hoa *= 1 + s.cost_growth
            ins *= 1 + s.cost_growth
            util *= 1 + s.cost_growth
            r_util *= 1 + s.cost_growth
            r_park *= 1 + s.cost_growth
            r_ins *= 1 + s.cost_growth
            room *= 1 + s.rent_growth
            mine *= 1 + s.rent_growth
    return pd.DataFrame(rows)


def first_month(s: Scenario) -> dict:
    loan = s.price * (1 - s.down_pct)
    pmt = monthly_payment(loan, s.mortgage_rate, s.term_years) if loan > 0 else 0.0
    costs = {
        "mortgage": pmt,
        "property tax": s.price * s.property_tax_rate / 12,
        "HOA": s.hoa_monthly,
        "insurance": s.insurance_annual / 12,
        "maintenance": s.price * s.maintenance_pct / 12,
        "utilities (whole unit)": s.utilities_total,
        "mortgage insurance": loan * s.pmi_rate / 12 if s.down_pct < 0.2 else 0.0,
    }
    income = {
        "roommate rent (after vacancy)": s.roommate_rent * (1 - s.vacancy),
        "roommate's share of utilities": s.utilities_total * s.roommate_util_share,
    }
    net_own = sum(costs.values()) - sum(income.values())
    net_rent = s.own_bedroom_rent + s.renter_utilities + s.renter_parking + s.renters_insurance_annual / 12
    return {"costs": costs, "income": income, "net_own": net_own, "net_rent": net_rent}


def sensitivity(s, horizons=(2, 4, 6, 8), appreciations=(-0.05, -0.02, 0.0, 0.02, 0.05)):
    out = {}
    for a in appreciations:
        df = simulate(replace(s, appreciation=a, years=max(horizons)))
        out[f"{a:+.0%} per year"] = [df.loc[df.year == h, "buy_advantage"].iloc[0] for h in horizons]
    return pd.DataFrame(out, index=[f"{h} yrs" for h in horizons]).T


if __name__ == "__main__":
    s = Scenario()
    fm = first_month(s)
    print("Month 1, buying the 2x2 with a roommate")
    for k, v in fm["costs"].items():
        print(f"  {k:<32}${v:>7,.0f}")
    for k, v in fm["income"].items():
        print(f"  less {k:<27}${v:>7,.0f}")
    print(f"  {'NET cost to the family':<32}${fm['net_own']:>7,.0f}   vs renting a bedroom ${fm['net_rent']:,.0f}\n")

    df = simulate(s)
    print(df.loc[df.year.isin([2, 4, 6, 10]), ["year", "buy_net_worth", "rent_net_worth", "buy_advantage"]]
            .round(0).to_string(index=False))
    print("\nBuy advantage ($), positive = buying wins:")
    print(sensitivity(s).round(0).to_string())

    print("\nAll-cash purchase:")
    dc = simulate(replace(s, down_pct=1.0))
    print(dc.loc[dc.year.isin([2, 4, 6]), ["year", "buy_advantage"]].round(0).to_string(index=False))
