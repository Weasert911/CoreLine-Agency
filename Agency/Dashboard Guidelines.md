# Enterprise Metrics – Calculation Reference
**Purpose:** Defines exact financial, sales, and operational calculation logic for the Agency Dashboard.  
**Currency Assumption:** `$` (full values, no thousands scaling)

---

# 1️⃣ Revenue Metrics

## Current Month Revenue

**Definition:**  
Sum of `Amount` from **_DEALS_** where:

- `Deal Status = Closed`
    
- `Payment Status = Paid`
    
- `Date` falls within the current calendar month
    

**Formula:**

```
Sum(Deals.Amount where Closed + Paid + Current Month)
```

---

## Previous Month Revenue

Sum of `Amount` from **_DEALS_** where:

- `Deal Status = Closed`
    
- `Payment Status = Paid`
    
- `Date` falls within previous calendar month
    

---

## Revenue Growth (%)

```
(Current Month Revenue − Previous Month Revenue)
÷ Previous Month Revenue × 100
```

If previous month revenue = 0 → Growth = 0 (avoid division error).

---

## Total Revenue (Lifetime)

Sum of `Amount` from **_DEALS_** where:

- `Deal Status = Closed`
    
- `Payment Status = Paid`
    

---

# 2️⃣ Cost & Profit Metrics

## Editor Cost (Monthly)

Sum of `Editor Cost` from **_DEALS_** where:

- `Deal Status = Closed`
    
- `Payment Status = Paid`
    
- `Date` is within current month
    

---

## Other Cost (Monthly)

Sum of `Other Cost` from **_DEALS_**  
Filtered by current month + Closed + Paid

---

## Monthly Operating Expenses

Sum of `Amount` from **_EXPENSES_** where:

- `Date` is within current month
    

---

## Commission Cost

For each qualifying deal:

```
Deal Amount × (Team Member Commission %)
```

Commission % is pulled from **_TEAM_** table.

Only applied if:

- `Status = Active`
    
- Team member matches `Outreacher` field
    

---

## Gross Profit

```
Current Month Revenue
− Editor Cost
− Other Cost
```

---

## Net Profit

```
Gross Profit
− Monthly Operating Expenses
− Commission Cost
```

---

## Profit Margin (%)

```
Net Profit ÷ Current Month Revenue × 100
```

If revenue = 0 → Margin = 0.

---

# 3️⃣ Sales & Lead Metrics

## Open Leads

Count of rows in **_LEADS_** where:

- `Status = Open`
    

---

## Closed Leads

Count of rows in **_LEADS_** where:

- `Status = Closed`
    

---

## Lead-to-Deal Conversion (%)

```
Number of Closed Deals ÷ Total Leads × 100
```

Closed Deals counted from **_DEALS_** where:

- `Deal Status = Closed`
    

---

# 4️⃣ Client Metrics

## Total Clients

Total number of rows in **_CLIENTS_**

---

## Active Clients

Count of rows in **_CLIENTS_** where:

- `Status = Active`
    

---

## Revenue Per Active Client

```
Total Revenue ÷ Active Clients
```

---

## Top 3 Client Revenue Concentration (%)

Measures dependency risk.

Steps:

1. Calculate total revenue per `Client ID`
    
2. Sort descending
    
3. Sum top 3
    
4. Divide by Total Revenue × 100
    

---

# 5️⃣ Risk & Stability Metrics

## Unpaid Revenue Exposure (%)

Sum of `Amount` where:

- `Deal Status = Closed`
    
- `Payment Status ≠ Paid`
    

Divided by:

```
Total Revenue × 100
```

Indicates cash flow risk.

---

## Operational Leverage Ratio

```
Current Month Revenue
÷ (Monthly Operating Expenses + Monthly Editor Cost)
```

Measures scalability efficiency.

Higher = better operational efficiency.

---

## Revenue Per Active Team Member

```
Current Month Revenue ÷ Number of Active Team Members
```

Active members counted from **_TEAM_** where:

- `Status = Active`
    

---

# 6️⃣ Performance Metrics

## Manager Revenue Ownership

Revenue grouped by:

- `Manager` field in **_DEALS_**
    

Filtered by:

- `Deal Status = Closed`
    
- `Payment Status = Paid`
    

---

## Outreach Performance

Revenue grouped by:

- `Outreacher` field in **_DEALS_**
    

Commission:

```
Deal Amount × Commission %
```

(Commission % pulled from TEAM table)

---

# 7️⃣ 12-Month Revenue Trend

Revenue grouped by:

- Year
    
- Month
    

Using:

- `Date` field from **_DEALS_**
    
- Only `Closed + Paid` deals
    

Used for:

- Growth tracking
    
- Seasonality detection
    
- Strategic forecasting
    

---

# Database Logic Relationships

## CLIENTS ↔ DEALS

Linked by:

```
Client ID
```

---

## LEADS ↔ DEALS

Linked by:

```
Lead ID
```

---

## TEAM ↔ DEALS

Linked by:

- `Outreacher`
    
- `Manager`
    

---

## EXPENSES

Standalone cost layer  
Filtered by month for profit calculations.

---

## Payroll

Currently excluded from Net Profit.

If added:

- Can be integrated into Monthly Operating Expenses
    
- Or separated into Fixed Cost layer for advanced margin tracking
    

---

# Governance Rules

1. Revenue is recognized only when:
    
    - Deal Status = Closed
        
    - Payment Status = Paid
        
2. All calculations are monthly calendar based.
    
3. Dashboard must always:
    
    - Avoid divide-by-zero errors
        
    - Validate date format (YYYY-MM-DD)
        
4. No manual overrides inside dashboard calculations.
    

---
