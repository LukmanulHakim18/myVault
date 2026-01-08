# MRG Architecture - Design Documents

## 📐 Design Documents Index

### Active Design Docs

#### 1. Pickup & Dropoff Point Accuracy Improvement
**Path**: `pickup-dropoff-accuracy/`  
**Status**: ✅ Implemented  
**Service**: Booking Service

Auto-snap system untuk meningkatkan akurasi pickup dan dropoff point menggunakan frequency-based dan popularity-based snapping.

**Documents:**
- [Overview & README](pickup-dropoff-accuracy/README.md)
- [C4 Design](pickup-dropoff-accuracy/c4-design.md)
- [Spike Analysis](pickup-dropoff-accuracy/spike-analysis.md)
- [Diagrams](pickup-dropoff-accuracy/diagrams/)

**Impact:**
- 50% reduction in incorrect pickup points
- 2-3 minutes faster pickup time
- 80%+ user acceptance rate

---

## 📁 Structure

```
design-docs/
├── pickup-dropoff-accuracy/    ✅ Auto-snap feature
│   ├── README.md
│   ├── c4-design.md
│   ├── spike-analysis.md
│   └── diagrams/
└── [future design docs...]
```

---

## 🎯 Design Doc Guidelines

### When to Create a Design Doc

Create a design doc for:
- ✅ New features with significant architectural impact
- ✅ Major system changes or refactoring
- ✅ Cross-service integrations
- ✅ Performance optimization initiatives
- ✅ Security or compliance changes

### Design Doc Template

Each design doc should include:
1. **README.md** - Overview, problem, solution, status
2. **Architecture diagrams** - C4, sequence, component diagrams
3. **Spike analysis** - Technical investigation (if needed)
4. **Implementation notes** - Key decisions, learnings

---

## 🔗 Related

- [RFCs](../rfcs/) - Request for Comments for major decisions
- [ADRs](../adrs/) - Architecture Decision Records
- [Services](../../02-services/) - Service implementations

---

**Last Updated**: 2025-01-03