## 🔧 Backend & Self-Test

- **[BE][Self-Test]** Return success jika sudah ada data tracking/state pada routine manager & tracker service  
  https://bluebirdgroup.clickup.com/t/9018711461/MYBB-4422  
  _Erik Rio Setiawan — Done (Development)_

- **[BE]** Add retry mechanism ketika gagal hit UPG (prevent order gantung)  
  https://bluebirdgroup.clickup.com/t/9018711461/MYBB-3397  
  _Erik Rio Setiawan — Done (Development)_

- **[BE]** Remove hyperlink address di email receipt  
  https://bluebirdgroup.clickup.com/t/9018711461/MYBB-3383  
  _Erik Rio Setiawan — Done (Development)_

- **[BE]** Reset Redis & database Autosnap (cleanup production setelah bugfix)  
  https://bluebirdgroup.clickup.com/t/9018711461/MYBB-3336  
  _Alfian Maulana Malik — Done_

- **[BE]** Scheduler untuk hapus expired frequent location  
  https://bluebirdgroup.clickup.com/t/9018711461/MYBB-3335  
  _Erik Rio Setiawan — Done (Development)_

- **[BE][Self-Test]** Add debug log saat error di routine manager  
  https://bluebirdgroup.clickup.com/t/9018711461/MYBB-3217  
  _Erik Rio Setiawan — Done (Development)_

- **[BE]** Order tamu stuck di finding driver & tidak bisa cancel  
  https://bluebirdgroup.clickup.com/t/9018711461/MYBB-3074  
  _Kamal Firdaus — Done_

- **[BE]** Improve SQL nearby search  
  https://bluebirdgroup.clickup.com/t/9018711461/MYBB-2940  
  _Erik Rio Setiawan — Done (Development)_

- **[BE]** Penambahan `trip_id` pada setter flow  
  https://bluebirdgroup.clickup.com/t/9018711461/MYBB-2888  
  _Erik Rio Setiawan — Done (Development)_

---

## 🧹 Tech Debt – RGR

- **[Tech Debt-RGR][BE]** Error mismatch response body vs DB (epayment_token)  
  https://bluebirdgroup.clickup.com/t/9018711461/MYBB-2869  
  _M Naufal Rifqi Ramdhani — Done (Development)_

- **[Tech Debt-RGR][BE]** Error mismatch locked fare saat POST order  
  https://bluebirdgroup.clickup.com/t/9018711461/MYBB-2865  
  _M Naufal Rifqi Ramdhani — Done (Development)_

---

## 🧪 Testing & Quality

- **[BE][Self-Test]** Add replica di Geo-service  
  https://bluebirdgroup.clickup.com/t/9018711461/MYBB-2837  
  _Erik Rio Setiawan — Done_

- **[BE][Self-Test]** Investigate automation test error STG/RGR  
  https://bluebirdgroup.clickup.com/t/9018711461/MYBB-2714  
  _Erik Rio Setiawan — Done (Development)_

- **[BE][Self-Test]** Improve unit test map-service  
  https://bluebirdgroup.clickup.com/t/9018711461/MYBB-2710  
  _M Naufal Rifqi Ramdhani — Done_

- **[BE][Self-Test]** Unit test snap pickup point (geo-service)  
  https://bluebirdgroup.clickup.com/t/9018711461/MYBB-2707  
  _Erik Rio Setiawan — Done (Development)_

---

## ⚙️ Performance, Cache & Infrastruktur

- **[BE]** Log before/after surcharge advance order  
  https://bluebirdgroup.clickup.com/t/9018711461/MYBB-2401  
  _Isfan — Done_

- **[BE]** Improve location registry cache (Halim border issue)  
  https://bluebirdgroup.clickup.com/t/9018711461/MYBB-2276  
  _Erik Rio Setiawan — Ready to Deploy_

- **[BE]** Migrate Location Registry ke Golang (Enera template)  
  https://bluebirdgroup.clickup.com/t/9018711461/MYBB-2123  
  _Axel Rori — Done_

- **[BE]** Separate Redis reads menggunakan replica (microservice)  
  https://bluebirdgroup.clickup.com/t/9018711461/MYBB-1937  
  _Erik Rio Setiawan — Done (Development)_

- **[BE]** Separate Redis reads untuk GET/HGET  
  https://bluebirdgroup.clickup.com/t/9018711461/MYBB-1936  
  _Erik Rio Setiawan — Ready to Deploy_

---

## 🚕 Order, Payment & Notification

- **[BE]** Handle zero value fleet response (Rent & Delivery)  
  https://bluebirdgroup.clickup.com/t/9018711461/MYBB-1975  
  _Alif Ahsanil Satria — Done (Development)_

- **[BE]** Handle zero value fleet response (Ride)  
  https://bluebirdgroup.clickup.com/t/9018711461/MYBB-1282  
  _Isfan — Ready to Deploy_

- **[BE]** Investigate & fix high latency get last order  
  https://bluebirdgroup.clickup.com/t/9018711461/MYBB-985  
  _Eko Nugroho — Done_

- **[BE]** Investigate & fix high latency get active order (Huawei)  
  https://bluebirdgroup.clickup.com/t/9018711461/MYBB-984  
  _Eko Nugroho — Ready to Deploy_

- **[BE]** Handle failover push HO setelah order created  
  https://bluebirdgroup.clickup.com/t/9018711461/MYBB-974  
  _Kamal Firdaus — Done_

- **[BE]** Investigate missing push notification Cititrans  
  https://bluebirdgroup.clickup.com/t/9018711461/MYBB-972  
  _Kamal Firdaus — Ready to Deploy_

---

## 📦 Cititrans & Legacy

- **[BE]** Refactor Code Cititrans Part 1  
  https://bluebirdgroup.clickup.com/t/9018711461/MYBB-966  
  _Lukmanul Hakim — Ready to Deploy_

- **[BE]** Create tech documentation Cititrans (Create Order Flow)  
  https://bluebirdgroup.clickup.com/t/9018711461/MYBB-803  
  _Kamal Firdaus — Done_

- **[BE][Self-Test]** Migrate Goldenbird calculation ke fare service  
  https://bluebirdgroup.clickup.com/t/9018711461/MYBB-603  
  _Alif Ahsanil Satria, Dimas Ilham Danesamarruf — Done_
