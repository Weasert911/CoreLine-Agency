# Enterprise Metrics – Calculation Reference

## Revenue Metrics

**Current Month Revenue**  
Sum of `Amount` from _**DEALS**_ where:

- `Deal Status = Closed`  
    
- `Date` is within current month  
    

Formula:  
Sum(Deals.Amount where Closed & current month)

---

**Previous Month Revenue**  
Sum of `Amount` from _**DEALS**_ where:

- `Deal Status = Closed`  
    
- `Date` is within previous month  
    

---

**Revenue Growth (%)**  
(Current Month Revenue − Previous Month Revenue) ÷ Previous Month Revenue × 100

---

**Total Revenue (Lifetime)**  
Sum of `Amount` from all _**DEALS**_ where:

- `Deal Status = Closed`  
    

---

## Cost & Profit Metrics

**Editor Cost (Monthly)**  
Sum of `Editor Cost` from _**DEALS**_ where:

- `Deal Status = Closed`  
    
- `Date` is current month  
    

---

**Other Cost (Monthly)**  
Sum of `Other Cost` from _**DEALS**_ (current month only)

---

**Monthly Expenses**  
Sum of `Amount` from _**EXPENSES**_ where:

- `Date` is current month  
    

---

**Commission Cost**  
For each deal:  
Deal Amount × (Team Member Commission %)

Commission % is taken from _**TEAM**_ table.  
Only applied if `Status = Active`.

---

**Gross Profit**  
Current Month Revenue − Editor Cost − Other Cost

---

**Net Profit**  
Gross Profit − Monthly Expenses − Commission Cost

---

**Profit** **Margin (%)**  
Net Profit ÷ Current Month Revenue × 100

---

## Sales & Lead Metrics

**Open Leads**  
Count of entries in _**LEADS**_ where `Status = Open`

---

**Closed Leads**  
Count of entries in _**LEADS**_ where `Status = Closed`

---

**Lead to Deal Conversion (%)**  
Number of Closed Deals ÷ Total Leads × 100

Deals counted from _**DEALS**_ where `Deal Status = Closed`

---

## Client Metrics

**Total Clients**  
Total rows in _**CLIENTS**_

---

**Active Clients**  
Count of rows in _**CLIENTS**_ where `Status = Active`

---

**Revenue Per Active Client**  
Total Revenue ÷ Active Clients

---

**Top 3 Client Concentration (%)**  
Sum of revenue from top 3 clients ÷ Total Revenue × 100

Client revenue calculated using `Client ID` in _**DEALS**_

---

## Risk & Stability Metrics

**Unpaid Revenue Exposure (%)**  
Sum of `Amount` where:

- `Deal Status = Closed`  
    
- `Payment Status ≠ Paid`  
    

Divided by Total Revenue × 100

---

**Operational Leverage Ratio**  
Current Month Revenue ÷ (Monthly Expenses + Monthly Editor Cost)

Measures scalability efficiency.

---

**Revenue Per Employee**  
Current Month Revenue ÷ Number of Active Team Members

Team members counted from _**TEAM**_ where `Status = Active`

---

## Performance Metrics

**Manager Revenue Ownership**  
Total revenue grouped by `Manager` field in _**DEALS**_

---

**Outreach Performance**  
Revenue grouped by `Outreacher` in _**DEALS**_

Commission calculated using:  
Deal Amount × Commission % (from TEAM table)

---

## 12-Month Revenue Trend

Revenue grouped by:  
Year + Month from `Date` field in _**DEALS**_  
Only `Deal Status = Closed`

---

# Database Logic Relationships

CLIENTS ↔ DEALS  
Linked by: `Client ID`

LEADS ↔ DEALS  
Linked by: `Lead ID`

TEAM ↔ DEALS  
Linked by: `Outreacher` and `Manager`

EXPENSES  
Standalone cost table (monthly filtered)

PAYROLL  
Currently not included in profit calculation  
Can be integrated into fixed cost layer if needed