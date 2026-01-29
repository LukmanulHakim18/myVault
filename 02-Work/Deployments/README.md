# 📁 Deployments Folder

**Purpose:** Central repository for all deployment overtime tracking and records

---

## 📂 Folder Structure

```
02-Work/Deployments/
├── README.md (this file)
├── deployment-overtime-summary.md (Master Dashboard)
├── monthly-deployment-overtime-2026.md (2026 Monthly Summary)
├── 2026-01-deployment-log.md (January 2026 Details)
└── [Future monthly logs...]
```

---

## 📋 File Descriptions

### `deployment-overtime-summary.md`
**Master Dashboard** - Overview of ALL deployment overtime
- Overall statistics (total hours, deployments, success rate)
- Summary by team (UPG, MRG)
- Chronological list of all deployments
- Quick reference for compensation calculation

### `monthly-deployment-overtime-2026.md`
**2026 Annual Summary** - Monthly breakdown for the year
- Month-by-month deployment hours
- Team distribution
- Quarterly summaries
- Annual totals

### `2026-01-deployment-log.md`
**January 2026 Details** - Detailed log for the month
- Complete deployment information per date
- Pre-deployment checklists
- Participating status and roles
- Outcomes and notes
- Monthly statistics

---

## 🔄 Workflow

### Adding New Deployment
1. Update `2026-[MM]-deployment-log.md` with deployment details
2. Update `monthly-deployment-overtime-2026.md` monthly totals
3. Update `deployment-overtime-summary.md` master dashboard

### Monthly Rollover
Create new file: `2026-[MM]-deployment-log.md` following January template

### Annual Rollover
Create new file: `monthly-deployment-overtime-2027.md` for next year

---

## ⚠️ IMPORTANT REMINDERS

**FOR CLAUDE:**
- ✅ ALWAYS check this folder first when asked about deployments
- ✅ ALWAYS update ALL three levels (detail log → monthly → summary)
- ✅ NEVER create deployment files outside this folder
- ✅ Follow existing format and structure
- ✅ Keep year correct (2026, not 2025!)

**FOR LUKMANUL HAKIM:**
- Master dashboard: `deployment-overtime-summary.md`
- Current month detail: `2026-01-deployment-log.md`
- Add deployment info BEFORE H-day
- Update actual hours AFTER deployment
- Review monthly summary for compensation calculation

---

## 📊 Quick Stats Access

**Current Month:** January 2026  
**Total Overtime YTD:** 1.5 hours  
**Pending Deployments:** 1 (MRG - 2026-01-29)

**Last Updated:** 2026-01-28

---

## 🔗 Related Folders
- `../Teams/` - Team-specific documentation
- `../Meetings/` - Meeting notes that may reference deployments
- `../Architecture/` - Architecture decisions affecting deployments
