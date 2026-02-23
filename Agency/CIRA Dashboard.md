```dataviewjs
// =============================
// LOAD DATABASE
// =============================
const filePath = "Agency Database.md";
const content = await dv.io.load(filePath);

if (!content) {
    dv.paragraph("Database file could not be loaded.");
    return;
}

const lines = content.split("\n");

// =============================
// HELPERS
// =============================
function isValidDate(dateStr){
    return /^\d{4}-\d{2}-\d{2}$/.test(dateStr);
}

function parseDate(str){
    return new Date(str + "T00:00:00");
}

// =============================
// DATA STRUCTURES
// =============================
let clients = [];
let leads = [];
let deals = [];
let expenses = [];
let team = [];

let section = "";

// =============================
// PARSE DATABASE
// =============================
for (let line of lines){

    if (line.includes("# ***CLIENTS***")) section="clients";
    if (line.includes("# ***LEADS***")) section="leads";
    if (line.includes("# ***DEALS***")) section="deals";
    if (line.includes("# ***EXPENSES***")) section="expenses";
    if (line.includes("# ***TEAM***")) section="team";

    if (!line.startsWith("|") || line.includes("---")) continue;

    let p = line.split("|").map(x=>x.trim());

    // CLIENTS
    if (section==="clients" && p[1] !== "ID"){
        clients.push({
            id:p[1],
            name:p[2],
            manager:p[3],
            status:p[9]
        });
    }

    // LEADS
    if (section==="leads" && p[1] !== "Lead ID"){
        leads.push({
            id:p[1],
            date:isValidDate(p[2]) ? parseDate(p[2]) : null,
            assigned:p[7],
            status:p[8]
        });
    }

    // DEALS
    if (section==="deals" && p[1] !== "Deal ID"){
        deals.push({
            id:p[1],
            date:isValidDate(p[2]) ? parseDate(p[2]) : null,
            clientId:p[3],
            leadId:p[4],
            outreacher:p[5],
            manager:p[6],
            editor:p[7],
            amount:Number(p[10])||0,
            editorCost:Number(p[11])||0,
            otherCost:Number(p[12])||0,
            discount:Number(p[13])||0,
            tax:Number(p[14])||0,
            paymentStatus:p[15],
            dealStatus:p[16]
        });
    }

    // EXPENSES
    if (section==="expenses" && p[1] !== "Date"){
        expenses.push({
            date:isValidDate(p[1]) ? parseDate(p[1]) : null,
            amount:Number(p[5])||0,
            type:p[6]
        });
    }

    // TEAM
    if (section==="team" && p[1] !== "Name"){
        team.push({
            name:p[1],
            role:p[2],
            commission:Number(p[3])||0,
            base:Number(p[4])||0,
            status:p[6]
        });
    }
}

// =============================
// TIME REFERENCE
// =============================
let today = new Date();
let currentMonth = today.getMonth();
let currentYear = today.getFullYear();
let lastMonth = currentMonth - 1;

// =============================
// CALCULATIONS
// =============================
let closedDeals = deals.filter(d=>d.dealStatus==="Closed");
let paidDeals = deals.filter(d=>d.paymentStatus==="Paid");

let monthRevenue = 0;
let lastMonthRevenue = 0;
let totalRevenue = 0;

let totalEditorCost = 0;
let totalOtherCost = 0;

let revenueByOutreacher = {};
let revenueByManager = {};
let commissions = {};

for (let d of closedDeals){

    if (!d.date) continue;

    let m = d.date.getMonth();
    let y = d.date.getFullYear();

    totalRevenue += d.amount;
    totalEditorCost += d.editorCost;
    totalOtherCost += d.otherCost;

    if (m===currentMonth && y===currentYear)
        monthRevenue += d.amount;

    if (m===lastMonth && y===currentYear)
        lastMonthRevenue += d.amount;

    revenueByOutreacher[d.outreacher] = 
        (revenueByOutreacher[d.outreacher]||0) + d.amount;

    revenueByManager[d.manager] = 
        (revenueByManager[d.manager]||0) + d.amount;

    let member = team.find(t=>t.name===d.outreacher && t.status==="Active");
    if (member){
        commissions[d.outreacher] = 
            (commissions[d.outreacher]||0) + 
            (d.amount * (member.commission/100));
    }
}

// Expenses (Current Month)
let monthExpenses = expenses
.filter(e=>e.date && e.date.getMonth()===currentMonth && e.date.getFullYear()===currentYear)
.reduce((a,b)=>a+b.amount,0);

// Growth
let growth = lastMonthRevenue>0
    ? ((monthRevenue-lastMonthRevenue)/lastMonthRevenue)*100
    : 0;

// Profit
let totalCommissions = Object.values(commissions)
.reduce((a,b)=>a+b,0);

let netProfit = monthRevenue 
    - monthExpenses 
    - totalCommissions;

let profitMargin = monthRevenue>0
    ? (netProfit/monthRevenue)*100
    : 0;

// Client Metrics
let activeClients = clients.filter(c=>c.status==="Active").length;
let totalClients = clients.length;

// Lead Metrics
let openLeads = leads.filter(l=>l.status==="Open").length;
let closedLeads = leads.filter(l=>l.status==="Closed").length;

// =============================
// EXECUTIVE SUMMARY
// =============================
dv.header(1,"Coreline Agency Executive Dashboard");

dv.table(
["Metric","Value"],
[
["Total Clients", totalClients],
["Active Clients", activeClients],
["Open Leads", openLeads],
["Closed Leads", closedLeads],
["Total Closed Deals", closedDeals.length],
["Current Month Revenue", monthRevenue],
["Previous Month Revenue", lastMonthRevenue],
["Revenue Growth (%)", growth.toFixed(2)],
["Current Month Expenses", monthExpenses],
["Total Commissions", totalCommissions],
["Net Profit (Current Month)", netProfit],
["Profit Margin (%)", profitMargin.toFixed(2)]
]
);

// =============================
// OUTREACH PERFORMANCE
// =============================
dv.header(2,"Outreach Performance");

dv.table(
["Outreacher","Total Revenue","Commission Earned"],
Object.entries(revenueByOutreacher)
.sort((a,b)=>b[1]-a[1])
.map(([k,v])=>[k,v,commissions[k]||0])
);

// =============================
// MANAGER PERFORMANCE
// =============================
dv.header(2,"Manager Revenue Ownership");

dv.table(
["Manager","Total Revenue Managed"],
Object.entries(revenueByManager)
.sort((a,b)=>b[1]-a[1])
);
// =============================
// ADVANCED AGENCY ANALYTICS
// =============================

// Gross Profit (Revenue - Direct Costs)
let grossProfit = monthRevenue - totalEditorCost - totalOtherCost;

// Gross Margin
let grossMargin = monthRevenue > 0
    ? (grossProfit / monthRevenue) * 100
    : 0;

// Average Revenue Per Deal
let avgRevenuePerDeal = closedDeals.length > 0
    ? totalRevenue / closedDeals.length
    : 0;

// Revenue Per Active Client
let revenuePerClient = activeClients > 0
    ? totalRevenue / activeClients
    : 0;

// Lead to Deal Conversion Rate
let conversionRate = leads.length > 0
    ? (closedDeals.length / leads.length) * 100
    : 0;

// Customer Acquisition Cost Proxy
// (Using outreach commissions as acquisition cost model)
let acquisitionCost = totalCommissions;

// Revenue Concentration Risk
let topClientRevenue = 0;
let revenueByClient = {};

for (let d of closedDeals){
    revenueByClient[d.clientId] =
        (revenueByClient[d.clientId]||0) + d.amount;
}

topClientRevenue = Math.max(...Object.values(revenueByClient), 0);

let concentrationRisk = totalRevenue > 0
    ? (topClientRevenue / totalRevenue) * 100
    : 0;

// Outreacher Efficiency (Revenue per Commission Paid)
let outreachEfficiency = totalCommissions > 0
    ? totalRevenue / totalCommissions
    : 0;

// Manager Portfolio Size
let managerClientCount = {};
for (let c of clients){
    if (c.status === "Active"){
        managerClientCount[c.manager] =
            (managerClientCount[c.manager]||0) + 1;
    }
}

// Burn Rate (Monthly Operating Cost)
let burnRate = monthExpenses + totalCommissions;

// Runway (if no new revenue)
let runway = burnRate > 0
    ? netProfit / burnRate
    : 0;

// =============================
// ADVANCED METRICS TABLE
// =============================
dv.header(2,"Financial Performance Analytics");

dv.table(
["Metric","Value"],
[
["Gross Profit (Current Month)", grossProfit],
["Gross Margin (%)", grossMargin.toFixed(2)],
["Average Revenue Per Deal", avgRevenuePerDeal],
["Revenue Per Active Client", revenuePerClient],
["Lead to Deal Conversion (%)", conversionRate.toFixed(2)],
["Top Client Revenue Concentration (%)", concentrationRisk.toFixed(2)],
["Outreacher Efficiency Ratio", outreachEfficiency.toFixed(2)],
["Monthly Burn Rate", burnRate],
["Runway (Months Estimate)", runway.toFixed(2)]
]
);

// =============================
// MANAGER PORTFOLIO DISTRIBUTION
// =============================
dv.header(2,"Manager Portfolio Distribution");

dv.table(
["Manager","Active Clients Managed"],
Object.entries(managerClientCount)
.sort((a,b)=>b[1]-a[1])
);
```

