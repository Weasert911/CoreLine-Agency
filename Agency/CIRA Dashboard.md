```dataviewjs
// =====================================================
// ENTERPRISE AGENCY DASHBOARD (FIXED FOR YOUR DB)
// =====================================================

const filePath = "Agency Database.md";
const content = await dv.io.load(filePath);

if (!content) {
    dv.paragraph("Database file could not be loaded.");
    return;
}

const lines = content.split("\n");

function isValidDate(dateStr){
    return /^\d{4}-\d{2}-\d{2}$/.test(dateStr);
}
function parseDate(str){
    return new Date(str + "T00:00:00");
}
function safeLower(str){
    return str ? str.trim().toLowerCase() : "";
}

let clients=[], leads=[], deals=[], expenses=[], team=[];
let section="";

// ================= PARSE DATABASE =================
for (let line of lines){

    if (line.includes("CLIENTS")) section="clients";
    if (line.includes("LEADS")) section="leads";
    if (line.includes("DEALS")) section="deals";
    if (line.includes("EXPENSES")) section="expenses";
    if (line.includes("TEAM")) section="team";

    if (!line.startsWith("|") || line.includes("---")) continue;

    let p=line.split("|").map(x=>x.trim());

    if (section==="clients" && p[1]!=="ID"){
        clients.push({
            id:p[1],
            status:p[9]
        });
    }

    if (section==="leads" && p[1]!=="Lead ID"){
        leads.push({
            id:p[1],
            status:p[8]
        });
    }

    if (section==="deals" && p[1]!=="Deal ID"){
        deals.push({
            id:p[1],
            date:isValidDate(p[2])?parseDate(p[2]):null,
            clientId:p[3],
            outreacher:p[5],
            manager:p[6],
            amount:Number(p[10])||0,
            editorCost:Number(p[11])||0,
            otherCost:Number(p[12])||0,
            discount:Number(p[13])||0,
            tax:Number(p[14])||0,
            paymentStatus:p[15],
            dealStatus:p[16],
            paymentDate:isValidDate(p[17])?parseDate(p[17]):null
        });
    }

    if (section==="expenses" && p[1]!=="Date"){
        expenses.push({
            date:isValidDate(p[1])?parseDate(p[1]):null,
            amount:Number(p[5])||0
        });
    }

    if (section==="team" && p[1]!=="Name"){
        team.push({
            name:p[1],
            commission:Number(p[3])||0,
            status:p[6]
        });
    }
}

// ================= TIME =================
let today=new Date();
let currentMonth=today.getMonth();
let currentYear=today.getFullYear();

let lastMonth=currentMonth-1;
let lastMonthYear=currentYear;
if(lastMonth<0){ lastMonth=11; lastMonthYear=currentYear-1; }

// ================= CLOSED & PAID DEALS =================
let validDeals = deals.filter(d =>
    safeLower(d.dealStatus)==="closed" &&
    safeLower(d.paymentStatus)==="paid"
);

// ================= FINANCIALS =================
let monthRevenue=0,lastMonthRevenue=0;
let monthEditorCost=0,monthOtherCost=0;
let totalRevenue=0;

let revenueByManager={};
let commissions={};

for(let d of validDeals){

    if(!d.date) continue;

    let m=d.date.getMonth();
    let y=d.date.getFullYear();

    totalRevenue+=d.amount;

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

// ================= EXPENSES =================
let monthExpenses=expenses
.filter(e=>e.date && e.date.getMonth()===currentMonth && e.date.getFullYear()===currentYear)
.reduce((a,b)=>a+b.amount,0);

let totalCommissions=Object.values(commissions).reduce((a,b)=>a+b,0);

// ================= PROFIT =================
let grossProfit=monthRevenue-monthEditorCost-monthOtherCost;
let netProfit=grossProfit-monthExpenses-totalCommissions;
let profitMargin=monthRevenue>0?(netProfit/monthRevenue)*100:0;
let growth=lastMonthRevenue>0?((monthRevenue-lastMonthRevenue)/lastMonthRevenue)*100:0;

// ================= CLIENT METRICS =================
let activeClients=clients.filter(c=>safeLower(c.status)==="active").length;
let totalClients=clients.length;
let openLeads=leads.filter(l=>safeLower(l.status)==="open").length;

// ================= OUTPUT =================
dv.header(1,"Enterprise Agency Dashboard");

dv.table(
["Metric","Value"],
[
["Total Clients",totalClients],
["Active Clients",activeClients],
["Open Leads",openLeads],
["Current Month Revenue (₹k)",monthRevenue],
["Revenue Growth (%)",growth.toFixed(2)],
["Gross Profit (₹k)",grossProfit],
["Net Profit (₹k)",netProfit],
["Profit Margin (%)",profitMargin.toFixed(2)]
]
);

dv.header(2,"Manager Revenue Ownership");

dv.table(
["Manager","Revenue Managed (₹k)"],
Object.entries(revenueByManager).sort((a,b)=>b[1]-a[1])
);
```
```dataviewjs
// ===============================
// WEEKLY PAYOUT DASHBOARD (FINAL REAL FIX)
// ===============================

let content = await dv.io.load("Agency Database.md");

const WEEK_START = moment().startOf("week");
const WEEK_END = moment().endOf("week");

let rows = content.split("\n")
    .filter(r => r.trim().startsWith("|D"));

let outreacherRevenue = {};
let managerProfit = {};
let totalRevenue = 0;
let totalProfit = 0;

for (let row of rows) {

    // Remove first and last pipe properly
    let cols = row.trim().slice(1, -1).split("|").map(c => c.trim());

    if (cols.length < 17) continue;

    let date = cols[1];
    let clientId = cols[2];
    let leadId = cols[3];
    let outreacher = cols[4];
    let manager = cols[5];
    let amount = Number(cols[9]) || 0;
    let editorCost = Number(cols[10]) || 0;
    let otherCost = Number(cols[11]) || 0;
    let discount = Number(cols[12]) || 0;
    let tax = Number(cols[13]) || 0;
    let paymentStatus = cols[14];
    let dealStatus = cols[15];
    let paymentDate = cols[16];

    if (
        dealStatus === "Closed" &&
        paymentStatus === "Paid" &&
        moment(paymentDate, "YYYY-MM-DD", true).isValid() &&
        moment(paymentDate, "YYYY-MM-DD").isBetween(WEEK_START, WEEK_END, null, '[]')
    ) {

        let profit = amount - editorCost - otherCost - discount - tax;

        totalRevenue += amount;
        totalProfit += profit;

        outreacherRevenue[outreacher] = (outreacherRevenue[outreacher] || 0) + amount;
        managerProfit[manager] = (managerProfit[manager] || 0) + profit;
    }
}

// DISPLAY

dv.header(2, "Weekly Period");
dv.paragraph(`${WEEK_START.format("YYYY-MM-DD")} → ${WEEK_END.format("YYYY-MM-DD")}`);

dv.header(2, "Outreacher Payout");

let outreachTable = Object.entries(outreacherRevenue).map(([name,revenue])=>[
    name,
    revenue,
    revenue * 0.10
]);

if (outreachTable.length===0){
    dv.paragraph("No payouts this week.");
}else{
    dv.table(["Outreacher","Revenue ($)","Payout 10% ($)"], outreachTable);
}

dv.header(2, "Manager Payout");

let managerTable = Object.entries(managerProfit).map(([name,profit])=>[
    name,
    profit,
    profit * 0.10
]);

if (managerTable.length===0){
    dv.paragraph("No payouts this week.");
}else{
    dv.table(["Manager","Profit ($)","Payout 10% ($)"], managerTable);
}

dv.header(2, "Weekly Summary");

dv.paragraph(`
Total Revenue: $${totalRevenue.toFixed(2)}  
Total Profit: $${totalProfit.toFixed(2)}  
Total Outreacher Payout: $${(totalRevenue*0.10).toFixed(2)}  
Total Manager Payout: $${(totalProfit*0.10).toFixed(2)}
`);
```

