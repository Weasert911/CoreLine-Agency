```dataviewjs
// =============================
// LOAD DATABASE
// =============================
const filePath = "Agency Database.md";
const content = await dv.io.load(filePath);
const meta = dv.page(filePath);

if (!content) {
    dv.paragraph("Database file could not be loaded.");
    return;
}

const lines = content.split("\n");

// =============================
// HELPER FUNCTIONS
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
let revenue = [];
let expenses = [];
let outreachers = [];
let managers = [];
let clients = [];

let section = "";
let mode = "";

// =============================
// PARSE DATABASE
// =============================
for (let line of lines){

    if (line.includes("# 👥 Clients")) section="clients";
    if (line.includes("# 👨‍💼 Employees")) section="employees";
    if (line.includes("# 💰 Revenue Log")) section="revenue";
    if (line.includes("# 💸 Expense Log")) section="expenses";

    if (line.includes("## Outreachers")) { mode="outreach"; continue; }
    if (line.includes("## Team Managers")) { mode="manager"; continue; }

    // Clients
    if (section==="clients" && line.startsWith("|") && !line.includes("Client Name") && !line.includes("---")){
        let p=line.split("|").map(x=>x.trim());
        clients.push({
            name:p[1],
            manager:p[2],
            active:p[3]==="Yes"
        });
    }

    // Employees
    if (section==="employees" && line.startsWith("|") && !line.includes("Name") && !line.includes("---")){
        let p=line.split("|").map(x=>x.trim());
        if (mode==="outreach") outreachers.push(p[1]);
        if (mode==="manager") managers.push(p[1]);
    }

    // Revenue
    if (section==="revenue" && line.startsWith("|") && !line.includes("Date") && !line.includes("---")){
        let p=line.split("|").map(x=>x.trim());
        if (isValidDate(p[1])){
            revenue.push({
                date:parseDate(p[1]),
                client:p[2],
                amount:Number(p[3])||0,
                outreacher:p[4]
            });
        }
    }

    // Expenses
    if (section==="expenses" && line.startsWith("|") && !line.includes("Date") && !line.includes("---")){
        let p=line.split("|").map(x=>x.trim());
        if (isValidDate(p[1])){
            expenses.push({
                date:parseDate(p[1]),
                category:p[2],
                amount:Number(p[3])||0
            });
        }
    }
}

// =============================
// TIME REFERENCE
// =============================
let latestDate = new Date(Math.max(...revenue.map(r=>r.date)));
let month = latestDate.getMonth();
let year = latestDate.getFullYear();
let lastMonth = month - 1;

// =============================
// CALCULATIONS
// =============================
let monthRevenue=0, lastMonthRevenue=0;
let monthExpenses=0;
let revenueByOutreacher={};
let revenueByManager={};
let commissions={};

for (let r of revenue){

    let m=r.date.getMonth();
    let y=r.date.getFullYear();

    if (m===month && y===year){
        monthRevenue+=r.amount;
        commissions[r.outreacher]=(commissions[r.outreacher]||0)+(r.amount*0.10);
    }

    if (m===lastMonth && y===year)
        lastMonthRevenue+=r.amount;

    revenueByOutreacher[r.outreacher]=(revenueByOutreacher[r.outreacher]||0)+r.amount;

    let clientObj=clients.find(c=>c.name===r.client);
    if(clientObj){
        revenueByManager[clientObj.manager]=(revenueByManager[clientObj.manager]||0)+r.amount;
    }
}

for (let e of expenses){
    if (e.date.getMonth()===month && e.date.getFullYear()===year)
        monthExpenses+=e.amount;
}

let growth= lastMonthRevenue>0
    ? ((monthRevenue-lastMonthRevenue)/lastMonthRevenue)*100
    :0;

let managerPool=monthRevenue*0.10;
let totalCommissions=Object.values(commissions).reduce((a,b)=>a+b,0);
let netProfit=monthRevenue-monthExpenses-managerPool-totalCommissions;
let profitMargin=monthRevenue>0?(netProfit/monthRevenue)*100:0;

let activeClients=clients.filter(c=>c.active);
let inactiveClients=clients.filter(c=>!c.active);

// =============================
// EXECUTIVE SUMMARY
// =============================
dv.header(1,"Coreline Agency – Executive Dashboard");

dv.table(
["Metric","Value"],
[
["Active Clients",activeClients.length],
["Inactive Clients",inactiveClients.length],
["Current Month Revenue",monthRevenue],
["Previous Month Revenue",lastMonthRevenue],
["Revenue Growth (%)",growth.toFixed(2)],
["Current Month Expenses",monthExpenses],
["Outreach Commissions (10%)",totalCommissions],
["Manager Revenue Allocation (10%)",managerPool],
["Net Profit",netProfit],
["Profit Margin (%)",profitMargin.toFixed(2)]
]
);

// =============================
// PERFORMANCE REPORTS
// =============================
dv.header(2,"Outreach Performance Summary");
dv.table(
["Outreacher","Total Revenue","Current Month Commission"],
Object.entries(revenueByOutreacher)
.sort((a,b)=>b[1]-a[1])
.map(([k,v])=>[k,v,commissions[k]||0])
);

dv.header(2,"Manager Revenue Allocation");
dv.table(
["Manager","Total Revenue Managed"],
Object.entries(revenueByManager).sort((a,b)=>b[1]-a[1])
);
```


# 📊 Coreline Agency — Dynamic Charts

```dataviewjs
const file = "Agency Database.md";
const content = await dv.io.load(file);
if (!content) { 
  dv.paragraph("Database file not found."); 
  return; 
}

const lines = content.split("\n");

let revenue = [];
let expenses = [];
let section = "";

// -----------------------------
// Helpers
// -----------------------------
function isDate(str){
  return /^\d{4}-\d{2}-\d{2}$/.test(str);
}

function getWeek(dateStr){
  const d = new Date(dateStr);
  const firstJan = new Date(d.getFullYear(),0,1);
  return Math.ceil((((d - firstJan) / 86400000) + firstJan.getDay()+1)/7);
}

// -----------------------------
// Parse Database
// -----------------------------
for (let line of lines){

  if (line.includes("# 💰 Revenue Log")) section="revenue";
  if (line.includes("# 💸 Expense Log")) section="expenses";

  if (section==="revenue" && line.startsWith("|") && !line.includes("Date") && !line.includes("---")){
    let p=line.split("|").map(x=>x.trim());
    if (isDate(p[1])){
      revenue.push({
        date:p[1],
        client:p[2],
        amount:Number(p[3]) || 0,
        outreacher:p[4]
      });
    }
  }

  if (section==="expenses" && line.startsWith("|") && !line.includes("Date") && !line.includes("---")){
    let p=line.split("|").map(x=>x.trim());
    if (isDate(p[1])){
      expenses.push({
        date:p[1],
        category:p[2],
        amount:Number(p[3]) || 0
      });
    }
  }
}

// -----------------------------
// Calculations
// -----------------------------

// Revenue by Month
let revenueByMonth = {};
revenue.forEach(r=>{
  let month = r.date.slice(0,7);
  revenueByMonth[month]=(revenueByMonth[month]||0)+r.amount;
});

// Revenue by Outreacher
let revenueByOut = {};
revenue.forEach(r=>{
  revenueByOut[r.outreacher]=(revenueByOut[r.outreacher]||0)+r.amount;
});

// Revenue by Client
let revenueByClient = {};
revenue.forEach(r=>{
  revenueByClient[r.client]=(revenueByClient[r.client]||0)+r.amount;
});

// Weekly Revenue
let revenueByWeek = {};
revenue.forEach(r=>{
  let week = "W"+getWeek(r.date);
  revenueByWeek[week]=(revenueByWeek[week]||0)+r.amount;
});

// Expenses by Category
let expenseByCat = {};
expenses.forEach(e=>{
  expenseByCat[e.category]=(expenseByCat[e.category]||0)+e.amount;
});

// Profit by Month
let expenseByMonth = {};
expenses.forEach(e=>{
  let month=e.date.slice(0,7);
  expenseByMonth[month]=(expenseByMonth[month]||0)+e.amount;
});

let profitByMonth={};
Object.keys(revenueByMonth).forEach(m=>{
  profitByMonth[m]=revenueByMonth[m]-(expenseByMonth[m]||0);
});

// -----------------------------
// Chart Renderer
// -----------------------------
function renderChart(type, labels, data, title){

  const colorPalette = [
    "#1F4E79",
    "#2E75B6",
    "#548235",
    "#A61C00",
    "#7030A0",
    "#BF9000",
    "#0F3057",
    "#375623",
    "#833C0C",
    "#44546A"
  ];

  const colors = labels.map((_, i) => `"${colorPalette[i % colorPalette.length]}"`);

  dv.paragraph(
    "```chart\n" +
    "type: " + type + "\n" +
    "labels: [" + labels.map(l => `"${l}"`).join(", ") + "]\n" +
    "series:\n" +
    "  - title: \"" + title + "\"\n" +
    "    data: [" + data.join(", ") + "]\n" +
    "    backgroundColor: [" + colors.join(", ") + "]\n" +
    "    borderColor: [" + colors.join(", ") + "]\n" +
    "```"
  );
}
// -----------------------------
// Render Charts
// -----------------------------

dv.header(2,"Monthly Revenue");
renderChart("bar",
  Object.keys(revenueByMonth),
  Object.values(revenueByMonth),
  "Revenue"
);

dv.header(2,"Monthly Profit");
renderChart("bar",
  Object.keys(profitByMonth),
  Object.values(profitByMonth),
  "Profit"
);

dv.header(2,"Weekly Revenue");
renderChart("line",
  Object.keys(revenueByWeek),
  Object.values(revenueByWeek),
  "Weekly Revenue"
);

dv.header(2,"Revenue by Outreacher");
renderChart("bar",
  Object.keys(revenueByOut),
  Object.values(revenueByOut),
  "Revenue"
);

dv.header(2,"Revenue by Client");
renderChart("bar",
  Object.keys(revenueByClient),
  Object.values(revenueByClient),
  "Revenue"
);

dv.header(2,"Expense Distribution");
renderChart("pie",
  Object.keys(expenseByCat),
  Object.values(expenseByCat),
  "Expenses"
);
```
