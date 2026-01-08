## Summary Tech Debt

### **Total Items**: 11 tech debts

---

## **Kategori Berdasarkan Service**

### **1. MPG2 Service (6 items)**

- **UPGN-3563**: Handle scheduler retry OVO
- **UPGN-3562**: Handle case gagal charge LinkAja di 1st attempt dengan debt = paid amount
- **UPGN-3561**: Explore impact jika ubah logic partial paid menjadi < paid amount (intermediate-ho)
- **UPGN-3272**: Handle Retry Transaction OVO with Fail Refund
- **UPGN-559**: LinkAja - Handle error 20091 (transaksi tidak bisa di-reverse karena sudah pernah di-reverse)
- **UPGN-3444**: Unifikasi logic outstanding untuk semua E-Wallet

### **2. E-Wallet Services (2 items)**

- **UPGN-3517**: (gopay-service) Improve start transaction
- **UPGN-3516**: (ovo-service) Improve start transaction

### **3. Scheduler (1 item)**

- **UPGN-3412**: Handle Event Push HO ECV dari Scheduler untuk menambahkan price component (platform fee, dll)

### **4. Email Service Migration (3 items)**

- **UPGN-3402**: (ECVPAY) Change email service to bbone
- **UPGN-3373**: (Payment Run) Change flow send email to bbone

---

## **Prioritas Berdasarkan Tema**

### **🔴 Critical - Payment & Transaction Handling**

- Retry mechanism OVO (scheduler & failed refund)
- LinkAja charging failures & reversal errors
- Partial payment logic impact analysis

### **🟡 Medium - Service Improvement**

- Start transaction optimization (GoPay & OVO)
- Outstanding logic unification untuk E-Wallet

### **🟢 Low - Infrastructure Migration**

- Email service migration ke bbone (ECVPAY & Payment Run)
- Scheduler enhancement untuk price components