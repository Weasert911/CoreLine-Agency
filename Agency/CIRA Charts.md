# Coreline Agency – Charts Dashboard

```dataviewjs

// =============================
// LOAD CHART.JS (CDN SAFE LOADER)
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
let team = [];

let section = "";

// =============================
// PARSE DATABASE
// =============================
for (let line of lines){

    if (line.includes("# ***DEALS***")) section="deals";
    if (line.includes("# ***EXPENSES***")) section="expenses";
    if (line.includes("# ***LEADS***")) section="leads";
    if (line.includes("# ***TEAM***")) section="team";

    if (!line.startsWith("|") || line.includes("---")) continue;

    let p = line.split("|").map(x=>x.trim());

    if (section==="deals" && p[1] !== "Deal ID"){
        deals.push({
            date:isValidDate(p[2]) ? parseDate(p[2]) : null,
            outreacher:p[5],
            manager:p[6],
            amount:Number(p[10])||0,
            paymentStatus:p[15],
            dealStatus:p[16]
        });
    }

    if (section==="expenses" && p[1] !== "Date"){
        expenses.push({
            date:isValidDate(p[1]) ? parseDate(p[1]) : null,
            amount:Number(p[5])||0
        });
    }

    if (section==="leads" && p[1] !== "Lead ID"){
        leads.push({ status:p[8] });
    }

    if (section==="team" && p[1] !== "Name"){
        team.push({
            name:p[1],
            commission:Number(p[3])||0,
            status:p[6]
        });
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
let commissions = {};

let closedDeals = deals.filter(d=>d.dealStatus==="Closed");

for (let d of closedDeals){

    if (!d.date) continue;

    let m = d.date.getMonth();
    let y = d.date.getFullYear();

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

for (let e of expenses){
    if (e.date && e.date.getMonth()===currentMonth && e.date.getFullYear()===currentYear)
        monthExpenses += e.amount;
}

let totalCommissions = Object.values(commissions)
.reduce((a,b)=>a+b,0);

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

dv.header(2,"Manager Revenue Ownership");
renderChart("bar",
    Object.keys(revenueByManager),
    Object.values(revenueByManager)
);

dv.header(2,"Profit Breakdown");
renderChart("pie",
    ["Revenue","Expenses","Commissions","Net Profit"],
    [monthRevenue, monthExpenses, totalCommissions, netProfit]
);

dv.header(2,"Lead Pipeline");
renderChart("pie",
    ["Open Leads","Closed Leads"],
    [openLeads, closedLeads]
);

```
