hari ini aku ada meeting Solusi FDS karena:
- bluebird kehilangan uang 1miliyar rupiah
- timeline 1 minggu

masukkan dari pak vp:
- prefentif
	- deteksi potensi
	- alert 
	- followoup
- reaktif
	- manual block
		- Devide id
		- BB id
	- auto block
		- BBid outstanding
		- device id outstanding : TODO
		- realation kartu dipakai dimana saja

## Fraude Case
order menggunakan feature EZ-Pay ini adalah (argo)
- pre auth 100k
- penggunaan 10 juta 
- preauth dibatalkan karena tidak cocok
- order outstanding



## Solution 
- BBD akan mengirimkan pengecekan N2C(nontunai to cash) ke mrg setiap 20s, mengirimkan actual_argo
- MRG(n2cservice) akan melakukan pengeceka order tersebut menggunakan pyament ewalet atau cc
- MRG(n2cservice) akan melakukan  pengecekan,
	- jika actual_argo melebihi dari nilai preauth terakhir maka MRG(n2cservice) akan melakukan cancel preauth ke UPG
	- MRG akan melakukan pre auth dengan actual argo yang telah ditambahkan dengan nila XT
	- jika preauth yang baru gagal maka mybb akan info ke BBD untuk melakukan 
