# ID Generator Project - Index

## Project Overview

Proyek **Two-Level ID Generation System** untuk platform transportasi Bluebird Group yang diimplementasikan pada Web Reservasi dan Rent Revamp MyBB.

**Status:** Draft (Awaiting Approval)  
**Created:** December 29, 2025  
**Platform:** Web Reservasi & MyBB Rent Revamp  

## Quick Access

### Main Documents

- [[ID-Generator-System-Overview]] - Executive summary dan overview lengkap
- [[ID-Generator-Database-Schema]] - Database design dan schema
- [[ID-Generator-API-Reference]] - Go API documentation
- [[ID-Generator-Implementation-Guide]] - Migration strategy dan best practices
- [[ID-Generator-Web-Reservasi-Rent-Revamp]] - Implementation details untuk Web Reservasi & Rent Revamp

## Quick Reference

### ID Format

**Booking ID:** `CCT-TTTTTHHHHHHHHH` (18 chars)
- CC = Channel (2 chars): MY, WB, CR, AP, CT, WA
- T = Type (1 char): S=Single, M=Multi
- TTTTT = Time Hash (5 chars base36)
- HHHHHHHH = DB Hash (8 chars)

**Order ID:** `CCPP-TTTTTHHHHHHHHH` (18 chars)
- CC = Channel (2 chars)
- PP = Product (2 chars): Vehicle+Service
- TTTTT = Time Hash (5 chars)
- HHHHHHHH = DB Hash (8 chars)

### Recognition Rule

Count characters before dash (-):
- **3 chars** → Booking ID
- **4 chars** → Order ID

### Common Products

**Web Reservasi:**
- BA = BlueBird Airport
- BR = BlueBird Ride
- SA = Silverbird Airport
- SR = Silverbird Ride

**Rent Revamp:**
- BH = BlueBird Hourly
- BL = BlueBird Daily
- SN = Silverbird Rent
- SL = Silverbird Daily

## Key Features

- ✅ User-friendly format (easy to communicate)
- ✅ Secure (XOR obfuscation)
- ✅ Analytics-ready (channel/product embedded)
- ✅ High performance (<0.3ms generation)
- ✅ Zero collisions (DB constraint)

## Implementation Status

### Completed
- ✅ RFC Documentation
- ✅ Database schema design
- ✅ Go package implementation
- ✅ Web Reservasi integration
- ✅ Rent Revamp MyBB integration

### In Progress
- 🔄 Customer support training
- 🔄 Frontend UI updates
- 🔄 Monitoring dashboard

### Planned
- ⏳ Multi-region support
- ⏳ QR code integration
- ⏳ Analytics dashboard

## Related Systems

- **Web Reservasi** - Web booking platform
- **MyBB** - Mobile application
- **Rent Revamp** - Rental service platform
- **Corporate Portal** - B2B booking
- **Call Center** - CS booking system
- **WhatsApp Bot** - Chat booking

## Architecture Highlights

```
Two-Level System:
  Booking (Parent)
    ├─ Order 1 (Child)
    ├─ Order 2 (Child)
    └─ Order N (Child)
```

## Performance Metrics

- **Generation Time:** <0.3ms average
- **Throughput:** 10,000+ IDs/second
- **Collision Rate:** 0%
- **Index Size:** ~50% smaller than UUID

## Local Files

RFC Documents:
- `D:\document\mybluebird\Architech\id generator\RFC_ID_GENERATOR.md`
- `D:\document\mybluebird\Architech\id generator\RFC_ID_GENERATOR_ID.md`

HTML Documentation:
- `D:\document\mybluebird\Architech\id generator\BOOKING_ID_SYSTEM.html`
- `D:\document\mybluebird\Architech\id generator\PRODUCT_SUMMARY.html`
- `D:\document\mybluebird\Architech\id generator\QUICK_REFERENCE.html`

## Tags

#architecture #id-generation #bluebird #transportation #system-design #web-reservasi #rent-revamp #mybb #index

---

**Last Updated:** 2025-12-31