# 🎯 OBSIDIAN MCP RULES - Quick Reference

**Purpose:** Compact rules untuk Claude saat menggunakan Obsidian MCP  
**Last Updated:** 2026-01-28

---

## 🚨 CRITICAL RULES (MUST FOLLOW)

1. **Deployments** → `02-Work/Deployments/` ONLY
   - Update 3 levels: detail log → monthly summary → master dashboard
   - Current month: `2026-01-deployment-log.md`

2. **Work Documentation** → `02-Work/` and appropriate subfolders
   - NOT in root, NOT in 01-Projects

3. **Personal Projects** → `01-Projects/` ONLY
   - Bluebird work ≠ Personal projects

4. **Team Specific**
   - MRG docs → `02-Work/Teams/MRG/`
   - UPG docs → `02-Work/Teams/UPG/`

---

## 📍 QUICK PATH REFERENCE

| Content Type | Correct Path |
|--------------|--------------|
| Deployment Overtime | `02-Work/Deployments/` |
| RFCs | `02-Work/Teams/{MRG\|UPG}/01-architecture/rfcs/` |
| ADRs | `02-Work/Teams/{MRG\|UPG}/01-architecture/adrs/` |
| Design Docs | `02-Work/Teams/{MRG\|UPG}/01-architecture/design-docs/` |
| Code Reviews | `02-Work/Code-Reviews/` |
| Meetings | `02-Work/Meetings/` |
| Go Programming | `02-Work/Go-Programming/` |
| Personal Notes | `07-Personal/` |

---

## ❌ COMMON MISTAKES TO AVOID

1. ❌ Creating deployment files in root or wrong folder
   ✅ Always use `02-Work/Deployments/`

3. ❌ Putting Bluebird work in `01-Projects/`
   ✅ Bluebird work goes to `02-Work/`

4. ❌ Forgetting to update all 3 deployment levels
   ✅ Update: detail → monthly → summary

5. ❌ Creating new top-level folders
   ✅ Use existing structure only

---

## 🔄 DEPLOYMENT WORKFLOW (MOST IMPORTANT)

**Location:** `02-Work/Deployments/`

**Files to Update (All 3 Levels):**
1. `2026-01-deployment-log.md` (Detail level - current month)
2. `monthly-deployment-overtime-2026.md` (Monthly summary)
3. `deployment-overtime-summary.md` (Master dashboard)

**Before Deployment:**
- Add deployment info to detail log

**After Deployment:**
- Update actual hours in detail log
- Update monthly totals in monthly summary
- Update master dashboard statistics

---

## 📚 FULL DOCUMENTATION

For complete structure and detailed guidelines:
- **Comprehensive Guide:** `VAULT-MASTER-INDEX.md`
- **Deployment Details:** `02-Work/Deployments/README.md`
- **Getting Started:** `README.md`

---

**Version:** 1.0  
**Created:** 2026-01-28  
**For:** Claude Assistant (MCP Operations)
