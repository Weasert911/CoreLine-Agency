```dataviewjs
// =====================================================
// ENTERPRISE AGENCY DASHBOARD (MONTHLY CORE ONLY)
// =====================================================

// =====================================================
// LOAD DATABASE FILE
// =====================================================
const filePath = "Agency Database.md";
const content = await dv.io.load(filePath);

if (!content) {
    dv.paragraph("Database file could not be loaded.");
    return;
}

const lines = content.split("\n");

// =====================================================
// HELPERS
// =====================================================
function isValidDate(dateStr){
    return /^\d{4}-\d{2}-\d{2}$/.test(dateStr);
}

function parseDate(str){
    return new Date(str + "T00:00:00");
}

function safeLower(str){
    return str ? str.trim().toLowerCase() : "";
}

// =====================================================
// DATA STRUCTURES
// =====================================================
let clients=[], leads=[], deals=[], expenses=[], team=[];
let section="";

// =====================================================
// PARSE DATABASE
// =====================================================
for (let line of lines){

    if (line.includes("# ***CLIENTS***")) section="clients";
    if (line.includes("# ***LEADS***")) section="leads";
    if (line.includes("# ***DEALS***")) section="deals";
    if (line.includes("# ***EXPENSES***")) section="expenses";
    if (line.includes("# ***TEAM***")) section="team";

    if (!line.startsWith("|") || line.includes("---")) continue;

    let p=line.split("|").map(x=>x.trim());

    if (section==="clients" && p[1]!=="ID"){
        clients.push({
            id:p[1],
            name:p[2],
            manager:p[3],
            status:p[9]
        });
    }

    if (section==="leads" && p[1]!=="Lead ID"){
        leads.push({
            id:p[1],
            date:isValidDate(p[2])?parseDate(p[2]):null,
            assigned:p[7],
            status:p[8]
        });
    }

    if (section==="deals" && p[1]!=="Deal ID"){
        deals.push({
            id:p[1],
            date:isValidDate(p[2])?parseDate(p[2]):null,
            clientId:p[3],
            leadId:p[4],
            outreacher:p[5],
            manager:p[6],
            editor:p[7],
            amount:Number(p[10])||0,
            editorCost:Number(p[11])||0,
            otherCost:Number(p[12])||0,
            paymentStatus:p[15],
            dealStatus:p[16]
        });
    }

    if (section==="expenses" && p[1]!=="Date"){
        expenses.push({
            date:isValidDate(p[1])?parseDate(p[1]):null,
            amount:Number(p[5])||0,
            type:p[6]
        });
    }

    if (section==="team" && p[1]!=="Name"){
        team.push({
            name:p[1],
            role:p[2],
            commission:Number(p[3])||0,
            status:p[6]
        });
    }
}

// =====================================================
// TIME HANDLING
// =====================================================
let today=new Date();
let currentMonth=today.getMonth();
let currentYear=today.getFullYear();

let lastMonth=currentMonth-1;
let lastMonthYear=currentYear;

if(lastMonth<0){
    lastMonth=11;
    lastMonthYear=currentYear-1;
}

// =====================================================
// CLOSED DEALS
// =====================================================
let closedDeals = deals.filter(d =>
    safeLower(d.dealStatus) === "closed"
);

// =====================================================
// MONTHLY FINANCIALS
// =====================================================
let monthRevenue=0,lastMonthRevenue=0;
let monthEditorCost=0,monthOtherCost=0;
let totalRevenue=0;

let revenueByClient={};
let revenueByManager={};
let commissions={};

for(let d of closedDeals){

    if(!d.date) continue;

    let m=d.date.getMonth();
    let y=d.date.getFullYear();

    totalRevenue+=d.amount;

    revenueByClient[d.clientId]=(revenueByClient[d.clientId]||0)+d.amount;
    revenueByManager[d.manager]=(revenueByManager[d.manager]||0)+d.amount;

    let member=team.find(t=>t.name===d.outreacher && safeLower(t.status)==="active");
    if(member){
        commissions[d.outreacher]=(commissions[d.outreacher]||0)
        + (d.amount*(member.commission/100));
    }

    if(m===currentMonth && y===currentYear){
        monthRevenue+=d.amount;
        monthEditorCost+=d.editorCost;
        monthOtherCost+=d.otherCost;
    }

    if(m===lastMonth && y===lastMonthYear){
        lastMonthRevenue+=d.amount;
    }
}

// =====================================================
// EXPENSES
// =====================================================
let monthExpenses=expenses
.filter(e=>e.date && e.date.getMonth()===currentMonth && e.date.getFullYear()===currentYear)
.reduce((a,b)=>a+b.amount,0);

let totalCommissions=Object.values(commissions).reduce((a,b)=>a+b,0);

// =====================================================
// PROFIT STRUCTURE
// =====================================================
let grossProfit=monthRevenue-monthEditorCost-monthOtherCost;
let netProfit=grossProfit-monthExpenses-totalCommissions;
let profitMargin=monthRevenue>0?(netProfit/monthRevenue)*100:0;
let growth=lastMonthRevenue>0?((monthRevenue-lastMonthRevenue)/lastMonthRevenue)*100:0;

// =====================================================
// CLIENT + LEAD METRICS
// =====================================================
let activeClients=clients.filter(c=>safeLower(c.status)==="active").length;
let totalClients=clients.length;
let openLeads=leads.filter(l=>safeLower(l.status)==="open").length;

// =====================================================
// RISK METRICS
// =====================================================
let sortedClients=Object.entries(revenueByClient).sort((a,b)=>b[1]-a[1]);
let top3Revenue=sortedClients.slice(0,3).reduce((a,b)=>a+b[1],0);
let top3Risk=totalRevenue>0?(top3Revenue/totalRevenue)*100:0;

let unpaidExposure=closedDeals
.filter(d=>safeLower(d.paymentStatus)!=="paid")
.reduce((a,b)=>a+b.amount,0);

let unpaidPercent=totalRevenue>0?(unpaidExposure/totalRevenue)*100:0;

// =====================================================
// OPERATIONAL METRICS
// =====================================================
let operationalLeverage=
(monthExpenses+monthEditorCost)>0
? monthRevenue/(monthExpenses+monthEditorCost)
:0;

let activeTeam=team.filter(t=>safeLower(t.status)==="active").length;
let revenuePerEmployee=activeTeam>0?monthRevenue/activeTeam:0;

// =====================================================
// 12-MONTH TREND
// =====================================================
let monthlyRevenueMap={};

for(let d of closedDeals){
    if(!d.date) continue;
    let key=d.date.getFullYear()+"-"+String(d.date.getMonth()+1).padStart(2,"0");
    monthlyRevenueMap[key]=(monthlyRevenueMap[key]||0)+d.amount;
}

// =====================================================
// DASHBOARD OUTPUT
// =====================================================
dv.header(1,"Enterprise Agency Dashboard");

dv.table(
["Metric","Value"],
[
["Total Clients",totalClients],
["Active Clients",activeClients],
["Open Leads",openLeads],
["Current Month Revenue",monthRevenue],
["Revenue Growth (%)",growth.toFixed(2)],
["Gross Profit",grossProfit],
["Net Profit",netProfit],
["Profit Margin (%)",profitMargin.toFixed(2)]
]
);

dv.header(2,"Risk & Stability Metrics");

dv.table(
["Metric","Value"],
[
["Top 3 Client Concentration (%)",top3Risk.toFixed(2)],
["Unpaid Revenue Exposure (%)",unpaidPercent.toFixed(2)],
["Operational Leverage Ratio",operationalLeverage.toFixed(2)],
["Revenue Per Employee",revenuePerEmployee.toFixed(2)]
]
);

dv.header(2,"Manager Revenue Ownership");

dv.table(
["Manager","Revenue Managed"],
Object.entries(revenueByManager).sort((a,b)=>b[1]-a[1])
);

dv.header(2,"12-Month Revenue Trend");

dv.table(
["Month","Revenue"],
Object.entries(monthlyRevenueMap).sort().slice(-12)
);
```
```dataviewjs
// ===============================
// WEEKLY PAYOUT DASHBOARD
// ===============================

let content = await dv.io.load("Agency Database.md");

// Current week
const WEEK_START = moment().startOf("week");
const WEEK_END = moment().endOf("week");

// Extract DEALS section
let dealsSection = content.split("# ***DEALS***")[1];
if (!dealsSection) {
    dv.paragraph("No DEALS section found.");
    return;
}

let tableText = dealsSection.split("# ***EXPENSES***")[0];

let rows = tableText.split("\n")
    .filter(r => r.trim().startsWith("| D"));

let outreacherRevenue = {};
let managerProfit = {};
let totalRevenue = 0;
let totalProfit = 0;

for (let row of rows) {

    let cols = row.split("|").map(c => c.trim()).filter(c => c !== "");

    let dealStatus = cols[15];
    let paymentStatus = cols[14];
    let paymentDate = cols[16];

    if (
        dealStatus === "Closed" &&
        paymentStatus === "Paid" &&
        moment(paymentDate).isBetween(WEEK_START, WEEK_END, null, '[]')
    ) {

        let outreacher = cols[4];
        let manager = cols[5];

        let amount = Number(cols[9]) || 0;
        let editorCost = Number(cols[10]) || 0;
        let otherCost = Number(cols[11]) || 0;
        let discount = Number(cols[12]) || 0;
        let tax = Number(cols[13]) || 0;

        let profit = amount - editorCost - otherCost - discount - tax;

        totalRevenue += amount;
        totalProfit += profit;

        if (!outreacherRevenue[outreacher])
            outreacherRevenue[outreacher] = 0;

        if (!managerProfit[manager])
            managerProfit[manager] = 0;

        outreacherRevenue[outreacher] += amount;
        managerProfit[manager] += profit;
    }
}

// DISPLAY

dv.header(2, "Weekly Period");
dv.paragraph(`${WEEK_START.format("YYYY-MM-DD")} → ${WEEK_END.format("YYYY-MM-DD")}`);

dv.header(2, "Outreacher Payout (10% Revenue)");

let outreachTable = [];

for (let name in outreacherRevenue) {
    let revenue = outreacherRevenue[name];
    outreachTable.push([
        name,
        `$${revenue.toFixed(2)}`,
        `$${(revenue * 0.10).toFixed(2)}`
    ]);
}

if (outreachTable.length === 0) {
    dv.paragraph("No payouts this week.");
} else {
    dv.table(
        ["Outreacher", "Revenue ($)", "Payout 10% ($)"],
        outreachTable
    );
}

dv.header(2, "Manager Payout (10% Profit)");

let managerTable = [];

for (let name in managerProfit) {
    let profit = managerProfit[name];
    managerTable.push([
        name,
        `$${profit.toFixed(2)}`,
        `$${(profit * 0.10).toFixed(2)}`
    ]);
}

if (managerTable.length === 0) {
    dv.paragraph("No payouts this week.");
} else {
    dv.table(
        ["Manager", "Profit ($)", "Payout 10% ($)"],
        managerTable
    );
}

dv.header(2, "Weekly Summary");

dv.paragraph(`
Total Revenue: $${totalRevenue.toFixed(2)}  
Total Profit: $${totalProfit.toFixed(2)}  
Total Outreacher Payout: $${(totalRevenue * 0.10).toFixed(2)}  
Total Manager Payout: $${(totalProfit * 0.10).toFixed(2)}
`);
```

