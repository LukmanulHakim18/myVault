# Tech-Id

- Doc: https://apipoi.idtek.io/api/docs
- Token: 118ad5b6d0774c4fde7b80b1f8593513a6e39981

Data terkait hasil spike : https://bluebirdgroup365.sharepoint.com/:f:/s/ArchitectandTW/EvynlC9BPNFKtNUtPgWV7q8BY9hBqmw8z0hNqgCmBBCKog?e=UCJUX7

## Yang telah dilakukan

- Melakukan query data yangperoleh dari tim data, `data anomali sample huawei`, `popular request dropoff`, `popular request pickup`, `popular actual dropof`, `popular actual pickup`
- Melakukan rouping `data anomali sample huawei`, 4611 data menjadi 744 data, kemudian melakukan query ke data tersebut.

## Hasil

- masih menemukan `plus-code` pada alamat.
- alamat yang terindikasi landmark mirp penulisan dengan alamat `GMO` kita Ex :
  - Ex: Wisma Mulia, Jl. Gatot Subroto No.40 2 Lantai 27, RT.6/RW.1, Kuningan Bar., Kec. Mampang Prpt., Kota Jakarta Selatan, Daerah Khusus Ibukota Jakarta 12710, Indonesia
  - PIK Mall Avenue, Jl. Pantai Indah Kapuk Boulevard No.RT.6, RT.6/RW.2, Kamal Muara, Kecamatan Penjaringan, Jkt Utara, Daerah Khusus Ibukota Jakarta 14470, Indonesia
  - "MALL KELAPA GADING 3 – JL. Boulevard Raya Kelapa Gading Lantai 3 Food Court Unit 3K-05, RT.13/RW.18, Klp. Gading Tim., Kec. Klp. Gading, Jkt Utara, Daerah Khusus Ibukota Jakarta 14240, Indonesia
- `grouping_sample_actual_data_pic_data_pickup` :
  - "total_data": 743,
  - "total_data_plus_code": 163,
  - "total_in_percent": "21.94%"
- `grouping_sample_request_data_pi_data_pickup`:
  - "total_data": 743,
  - "total_data_plus_code": 142,
  - "total_in_percent": "19.11%"
- `popular_dropoff_data_pickup`:
  - "total_data": 100,
  - "total_data_plus_code": 17,
  - "total_in_percent": "17.00%"
- `popular_pickup_data_pickup`:
  - "total_data": 100,
  - "total_data_plus_code": 18,
  - "total_in_percent": "18.00%"
- `popular_request_dropoff_data_pickup`:
  - "total_data": 99,
  - "total_data_plus_code": 17,
  - "total_in_percent": "17.17%"
- `popular_request_pickup_data_pickup`:
  - "total_data": 100,
  - "total_data_plus_code": 14,
  - "total_in_percent": "14.00%"

## Yang perlu di tanyakan

- Perlu confirmasi kenapa masih ketemu plus code
- perlu confirmasi formating address terkait landmark dan subplace
