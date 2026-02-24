# Coreline Agency – Charts Dashboard

```dataviewjs
// =============================
// LOAD CHART.JS
// =============================
if (!window.Chart) {
    const script = document.createElement("script");
    script.src = "https://cdn.jsdelivr.net/npm/chart.js";
    document.head.appendChild(script);
    await new Promise(resolve => script.onload = resolve);
}

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
let deals = [];
let expenses = [];
let leads = [];

let section = "";

// =============================
// PARSE DATABASE
// =============================
for (let line of lines){

    if (line.includes("# ***DEALS***")) section="deals";
    if (line.includes("# ***EXPENSES***")) section="expenses";
    if (line.includes("# ***LEADS***")) section="leads";

    if (!line.startsWith("|") || line.includes("---")) continue;

    let p = line.split("|").map(x=>x.trim()).filter(x=>x!== "");

    // DEALS
    if (section==="deals" && p[0] !== "Deal ID"){

        deals.push({
            paymentDate:isValidDate(p[16]) ? parseDate(p[16]) : null,
            outreacher:p[4],
            manager:p[5],
            amount:Number(p[9])||0,
            editorCost:Number(p[10])||0,
            otherCost:Number(p[11])||0,
            discount:Number(p[12])||0,
            tax:Number(p[13])||0,
            paymentStatus:p[14],
            dealStatus:p[15]
        });
    }

    // EXPENSES
    if (section==="expenses" && p[0] !== "Date"){
        expenses.push({
            date:isValidDate(p[0]) ? parseDate(p[0]) : null,
            amount:Number(p[4])||0
        });
    }

    // LEADS
    if (section==="leads" && p[0] !== "Lead ID"){
        leads.push({ status:p[7] });
    }
}

// =============================
// TIME CONTEXT
// =============================
let today = new Date();
let currentMonth = today.getMonth();
let currentYear = today.getFullYear();
let lastMonth = currentMonth - 1;

// =============================
// CALCULATIONS
// =============================
let monthRevenue = 0;
let lastMonthRevenue = 0;
let monthExpenses = 0;

let revenueByOutreacher = {};
let revenueByManager = {};

let outreacherCommissions = 0;
let managerCommissions = 0;

// Only Paid + Closed deals
let validDeals = deals.filter(d =>
    d.dealStatus==="Closed" &&
    d.paymentStatus==="Paid" &&
    d.paymentDate
);

for (let d of validDeals){

    let m = d.paymentDate.getMonth();
    let y = d.paymentDate.getFullYear();

    let profit = d.amount - d.editorCost - d.otherCost - d.discount - d.tax;

    // Monthly Revenue
    if (m===currentMonth && y===currentYear)
        monthRevenue += d.amount;

    if (m===lastMonth && y===currentYear)
        lastMonthRevenue += d.amount;

    // Revenue by Outreacher
    revenueByOutreacher[d.outreacher] =
        (revenueByOutreacher[d.outreacher]||0) + d.amount;

    // Revenue by Manager (profit-based ownership)
    revenueByManager[d.manager] =
        (revenueByManager[d.manager]||0) + profit;

    // Commissions
    if (m===currentMonth && y===currentYear){
        outreacherCommissions += d.amount * 0.10;
        managerCommissions += profit * 0.10;
    }
}

// Expenses
for (let e of expenses){
    if (e.date && e.date.getMonth()===currentMonth && e.date.getFullYear()===currentYear)
        monthExpenses += e.amount;
}

let totalCommissions = outreacherCommissions + managerCommissions;

let netProfit = monthRevenue - monthExpenses - totalCommissions;

let openLeads = leads.filter(l=>l.status==="Open").length;
let closedLeads = leads.filter(l=>l.status==="Closed").length;

// =============================
// GENERIC CHART FUNCTION
// =============================
function renderChart(type, labels, data){

    let canvas = dv.el("canvas", "");
    let ctx = canvas.getContext("2d");

    new Chart(ctx,{
        type: type,
        data: {
            labels: labels,
            datasets: [{
                data: data
            }]
        },
        options: {
            responsive: true
        }
    });
}

// =============================
// CHARTS
// =============================

dv.header(2,"Revenue Comparison");
renderChart("bar",
    ["Last Month","Current Month"],
    [lastMonthRevenue, monthRevenue]
);

dv.header(2,"Outreacher Revenue");
renderChart("bar",
    Object.keys(revenueByOutreacher),
    Object.values(revenueByOutreacher)
);

dv.header(2,"Manager Profit Ownership");
renderChart("bar",
    Object.keys(revenueByManager),
    Object.values(revenueByManager)
);

dv.header(2,"Profit Breakdown (Current Month)");
renderChart("pie",
    ["Revenue","Expenses","Outreach Commissions","Manager Commissions","Net Profit"],
    [monthRevenue, monthExpenses, outreacherCommissions, managerCommissions, netProfit]
);

dv.header(2,"Lead Pipeline");
renderChart("pie",
    ["Open Leads","Closed Leads"],
    [openLeads, closedLeads]
);
```
