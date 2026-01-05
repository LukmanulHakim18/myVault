---
title: "Dashboard - Personal Knowledge Base"
tags: [dashboard, home]
---

# 📊 Dashboard - Personal Knowledge Base

*Last updated: {{date}}*

> Welcome back, Lukmanul! Ini adalah command center untuk semua dokumentasi dan knowledge Anda.

---

## 🎯 Quick Links

### Most Used
- [[00-Inbox/quick-notes]] - Catatan cepat
- [[daily-note-template]] - Daily notes
- [[rfc-template]] - Create new RFC
- [[code-review-template]] - Do code review

### Important Documents
- [[02-Work/Go-Programming/go-programming-moc]] - Go Programming MOC
- [[06-Career/goals-and-objectives]] - Career goals
- [[04-Resources/Templates/]] - All templates

---

## 📋 Active Projects

```dataview
TABLE 
  status as "Status",
  priority as "Priority", 
  deadline as "Deadline",
  progress as "Progress %"
FROM "01-Projects/Active"
SORT priority DESC, deadline ASC
```

**Quick Stats**:
- Active Projects: 
- Completed This Month: 
- On-Hold: 

---

## ✅ This Week's Focus

```dataview
TASK
WHERE !completed 
  AND contains(tags, "this-week")
  AND file.path != this.file.path
GROUP BY file.link
SORT priority DESC
```

---

## 🚧 Blocked Items

```dataview
TABLE 
  blocker as "Blocker",
  assigned_to as "Owner",
  blocked_since as "Since"
FROM "01-Projects"
WHERE status = "blocked"
SORT blocked_since ASC
```

---

## 📝 Recent Code Reviews

```dataview
TABLE 
  reviewee as "Developer",
  pr_link as "PR",
  date as "Date",
  status as "Status"
FROM "02-Work/Code-Reviews"
SORT date DESC
LIMIT 5
```

---

## 🔥 Technical Debt Tracker

```dataview
TABLE 
  severity as "Severity",
  effort as "Effort (days)",
  impact as "Impact",
  file.ctime as "Reported"
FROM "02-Work"
WHERE contains(tags, "tech-debt")
  AND status != "resolved"
SORT severity DESC, impact DESC
```

---

## 📚 Learning Progress

```dataview
TABLE 
  category as "Category",
  status as "Status",
  completion as "Progress %",
  file.mtime as "Last Update"
FROM "03-Learning"
WHERE status = "in-progress"
SORT file.mtime DESC
```

**This Month**:
- Books Reading: 
- Courses In Progress: 
- Articles Saved: 

---

## 🗓️ Upcoming Events

### This Week
- [ ] Meeting: [Topic] - [Date, Time]
- [ ] Deadline: [Task] - [Date]
- [ ] Review: [PR/RFC] - [Date]

### Next Week
- [ ] 
- [ ] 

---

## 📊 Current Sprint Metrics

**Sprint**: Sprint XX (Week 1 of 2)
- **Sprint Goal**: 
- **Completed Stories**: X / Y
- **Velocity**: X points
- **Burndown**: 🟢 On Track | 🟡 At Risk | 🔴 Behind

### Sprint Tasks
```dataview
TASK
WHERE contains(tags, "current-sprint") 
  AND !completed
GROUP BY status
```

---

## 💡 Ideas & Inbox

### Unprocessed Items (00-Inbox)
```dataview
LIST
FROM "00-Inbox"
WHERE file.name != "README"
SORT file.ctime DESC
LIMIT 10
```

**Total items in inbox**: 

⚠️ *Process inbox regularly - aim for inbox zero!*

---

## 📈 Personal Metrics

### This Month
- **Code Reviews**: X completed
- **RFCs Written**: Y
- **Bugs Fixed**: Z
- **Features Delivered**: A
- **Meeting Hours**: XX hrs
- **Deep Work Hours**: YY hrs

### Goals Progress
- [ ] Goal 1: XX% complete
- [ ] Goal 2: XX% complete
- [ ] Goal 3: XX% complete

---

## 🎓 Knowledge Areas

### Go Programming
- [[go-best-practices]]
- [[concurrency-patterns]]
- [[error-handling-patterns]]
- [[testing-strategies]]

### Architecture
- [[microservices-patterns]]
- [[event-driven-architecture]]
- [[system-design-principles]]

### Transportation Domain
- [[booking-systems]]
- [[routing-algorithms]]
- [[fleet-management]]

---

## 🔗 Quick Actions

### Create New
- [ ] [[New RFC|Create RFC]]
- [ ] [[New Code Review|Start Code Review]]
- [ ] [[New Meeting Notes|Log Meeting]]
- [ ] [[New Project|Start Project]]

### Review & Process
- [ ] Process Inbox (00-Inbox)
- [ ] Review Active Projects
- [ ] Update Career Goals
- [ ] Archive Completed Items

---

## 📞 Team Communication

### Recent Meetings
```dataview
TABLE 
  meeting_type as "Type",
  duration as "Duration",
  attendees as "Attendees"
FROM "05-Team/Meetings"
SORT file.ctime DESC
LIMIT 5
```

### Knowledge Sharing Sessions
```dataview
LIST
FROM "05-Team/Knowledge-Sharing"
SORT file.ctime DESC
LIMIT 5
```

---

## 🏆 Recent Achievements

```dataview
LIST
FROM "06-Career"
WHERE contains(tags, "achievement")
SORT file.ctime DESC
LIMIT 5
```

---

## 🔄 Weekly Review Checklist

**Last Review**: [Date]
**Next Review**: [Date]

- [ ] Review all active projects
- [ ] Process inbox to zero
- [ ] Update goals progress
- [ ] Archive completed items
- [ ] Plan next week priorities
- [ ] Review team communication
- [ ] Update skills matrix
- [ ] Reflect on learnings

---

## 💾 Backup Status

**Last Backup**: [Date]
**Backup Method**: Obsidian Git / Manual
**Status**: ✅ Up to date | ⚠️ Pending

---

## 📱 Quick Stats

**Vault Statistics**:
- Total Notes: XXX
- Total Links: XXX
- Daily Notes: XXX
- Last 7 Days Activity: XXX notes modified

**Most Active Tags**:
#go, #architecture, #meeting, #project

---

## 🎯 Focus Mode

**Current Focus**: [What you're working on right now]
**Started**: [Time]
**Goal**: [What you want to achieve]

**Pomodoro Tracker**: 🍅🍅🍅🍅

---

*Remember: Quality over quantity. Better fewer well-documented notes than many scattered ones.*

---

## Navigation

- [[README]] - Vault guide
- [[02-Work/Go-Programming/go-programming-moc]] - Go MOC
- [[06-Career/goals-and-objectives]] - Career planning