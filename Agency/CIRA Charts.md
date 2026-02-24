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

if (!content){
    dv.paragraph("Database file could not be loaded.");
    return;
}

const lines = content.split("\n");

// =============================
// HELPERS
// =============================
function isValidDate(d){ return /^\d{4}-\d{2}-\d{2}$/.test(d); }
function parseDate(d){ return new Date(d+"T00:00:00"); }
function safeLower(s){ return s ? s.trim().toLowerCase() : ""; }

// =============================
// DATA STRUCTURES
// =============================
let deals=[], expenses=[], leads=[], clients=[];
let section="";

// =============================
// PARSE DATABASE
// =============================
for (let line of lines){

    if (line.includes("DEALS")) section="deals";
    if (line.includes("EXPENSES")) section="expenses";
    if (line.includes("LEADS")) section="leads";
    if (line.includes("CLIENTS")) section="clients";

    if (!line.startsWith("|") || line.includes("---")) continue;

    let cols = line.trim().slice(1,-1).split("|").map(x=>x.trim());

    // DEALS
    if (section==="deals" && cols[0] !== "Deal ID"){
        deals.push({
            date:isValidDate(cols[1])?parseDate(cols[1]):null,
            paymentDate:isValidDate(cols[16])?parseDate(cols[16]):null,
            clientID:cols[2],
            outreacher:cols[4],
            manager:cols[5],
            amount:Number(cols[9])||0,
            editorCost:Number(cols[10])||0,
            otherCost:Number(cols[11])||0,
            discount:Number(cols[12])||0,
            tax:Number(cols[13])||0,
            paymentStatus:cols[14],
            dealStatus:cols[15]
        });
    }

    // EXPENSES
    if (section==="expenses" && cols[0] !== "Date"){
        expenses.push({
            date:isValidDate(cols[0])?parseDate(cols[0]):null,
            type:cols[5],
            amount:Number(cols[4])||0
        });
    }

    // LEADS
    if (section==="leads" && cols[0] !== "Lead ID"){
        leads.push({ status:cols[7] });
    }

    // CLIENTS
    if (section==="clients" && cols[0] !== "ID"){
        clients.push({
            id:cols[0],
            acquisition:cols[3],
            status:cols[8],
            country:cols[9],
            industry:cols[10]
        });
    }
}

// =============================
// FILTER VALID DEALS
// =============================
let validDeals = deals.filter(d =>
    safeLower(d.dealStatus)==="closed" &&
    safeLower(d.paymentStatus)==="paid"
);

// =============================
// MONTHLY TREND (LAST 6 MONTHS)
// =============================
let monthlyRevenue={};
let now=new Date();

for (let i=5;i>=0;i--){
    let d=new Date(now.getFullYear(), now.getMonth()-i,1);
    let key=d.getFullYear()+"-"+(d.getMonth()+1);
    monthlyRevenue[key]=0;
}

for (let d of validDeals){
    if (!d.paymentDate) continue;
    let key=d.paymentDate.getFullYear()+"-"+(d.paymentDate.getMonth()+1);
    if (monthlyRevenue[key]!==undefined)
        monthlyRevenue[key]+=d.amount;
}

// =============================
// REVENUE BY ACQUISITION
// =============================
let revenueByChannel={};

for (let d of validDeals){
    let client=clients.find(c=>c.id===d.clientID);
    if (!client) continue;
    revenueByChannel[client.acquisition]=(revenueByChannel[client.acquisition]||0)+d.amount;
}

// =============================
// CONVERSION RATE
// =============================
let totalLeads=leads.length;
let closedLeads=leads.filter(l=>safeLower(l.status)==="closed").length;
let conversionRate = totalLeads>0 ? ((closedLeads/totalLeads)*100).toFixed(1) : 0;

// =============================
// CLIENT STATUS DISTRIBUTION
// =============================
let clientStatusCount={};
for (let c of clients){
    clientStatusCount[c.status]=(clientStatusCount[c.status]||0)+1;
}

// =============================
// EXPENSE TYPE BREAKDOWN
// =============================
let expenseTypeCount={};
for (let e of expenses){
    expenseTypeCount[e.type]=(expenseTypeCount[e.type]||0)+e.amount;
}

// =============================
// GENERIC CHART
// =============================
function renderChart(type, labels, data){
    let canvas=dv.el("canvas","");
    let ctx=canvas.getContext("2d");

    new Chart(ctx,{
        type:type,
        data:{
            labels:labels,
            datasets:[{ data:data }]
        },
        options:{ responsive:true }
    });
}

// =============================
// DASHBOARD
// =============================

dv.header(2,"Revenue Trend (Last 6 Months)");
renderChart("line",
    Object.keys(monthlyRevenue),
    Object.values(monthlyRevenue)
);

dv.header(2,"Revenue by Acquisition Channel");
renderChart("bar",
    Object.keys(revenueByChannel),
    Object.values(revenueByChannel)
);

dv.header(2,"Lead Conversion Rate");
dv.paragraph("Total Leads: "+totalLeads);
dv.paragraph("Closed Leads: "+closedLeads);
dv.paragraph("Conversion Rate: "+conversionRate+"%");

dv.header(2,"Client Status Distribution");
renderChart("pie",
    Object.keys(clientStatusCount),
    Object.values(clientStatusCount)
);

dv.header(2,"Expense Breakdown");
renderChart("pie",
    Object.keys(expenseTypeCount),
    Object.values(expenseTypeCount)
);
```
