Untuk project tech debt yang kita fokuskan ada cqrs untuk all transaction, jadi sekarang upg memilik koneep begini.
upg mempunyai payment ewalet dan masing masing payment punya service sendiri untuk process transaksinya:
- ovo
- gopay
- dana
- shopeepay
- linkaja
- isaku
sedangkan transaksi menggunakan kartu menggunakan card payment service.

MPG2 menampung semua transaction fare walet dan card masuk ke sini

sedangkan transaksi yang tips dan extra masuk ke masing masing service sehingga ini terpisah.

aku mau merancang mekanisme cqrs pada transaction service untuk mencatat  semua transaction
kedepannya dia mempunyai dua table
- transaction as parent 
- transaction item as child(fare, extra, tips)
	- disini ada status yang menentukan ini masih gantung
	- masih punya sisa tagihan jika partialy paid
	- dan masing masing item bisa memiliki beberapa kali pembayaran.(concern) masih menjadi misteri record baru atau tetap satu record.
jadi si transaction service akan menerima event push rabbit untuk setiap ada order yang masuk sehingga ketika request order gantung tidak perlu lagi collect data ke semua service terkait.