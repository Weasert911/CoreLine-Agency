# Dashboard

## Overview
This dashboard provides a comprehensive view of agency operations, including team performance, sales pipeline, project status, finance, and legal tracking.

## Key Metrics
- Total Revenue: {{total_revenue}}
- Active Projects: {{active_projects}}
- Overdue Projects: {{overdue_projects}}
- Outstanding Invoices: {{outstanding_invoices}}

## Dataview Queries
### Recent Deals
```dataview
table date, value, status
from "Deals"
sort date desc
limit 10
```

### Active Projects
```dataview
table client, editor, deadline, status
from "Projects"
where status != "Delivered"
sort deadline asc
limit 10
```

### Editor Productivity
```dataview
table revisions, turnaround_time, videos_completed
from "Team"
where role = "editor"
sort videos_completed desc
limit 10
```
