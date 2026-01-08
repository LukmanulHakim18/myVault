# Spike Drop off and Pick up Point Accuracy Improvement `FLOW`

## Simpan data frekuensi pengguna untuk optimasi snap.

Untuk menyimpan data bersih dan siap pakai alangkah baiknya setiap order complete melakukan langkah langkah di bawah ini untuk keperluan featur.

- Mendengarkan event order complete dari BBD.
- cek ke `frequency place` user table and `popular public place`
- jika tidak ada maka cek apakah point tersebut masuk kedalam landmark apa tidak.
- simpan data ke table `frequency place` jika memenuhi syarat ketentuan
- simpan data ke table `popular public place` jika memenuhi syarat ketentuan dan jika sudah ada hanya tambahkan inc counter pada colom `pickedUp` atau `droppedOff`.

### Sequence

![alt-setter](setter.png)

## Implementasi auto-snap berdasarkan frekuensi (>=2 perjalanan).

Buat endpoint baru untuk rekomendasi frekuensi place, mengembalikan dua data `prifate_frequency_place` dan `popular_public_place`. Bertujuan agar tidak mengganggu current flow.
Mobile akan menerima data dan akan langsung menerapkan data yang diterima berdasarkan perioritas.
![alt-getter](../src/auto_snap/getter.png)

## Prioritaskan radius-based snapping (residential vs commercial).

## Clustering pickup/drop-off untuk Landmark dengan subplace.

# Spike Drop off and Pick up Point Accuracy Improvement `CONCERN`

- Penambahan beban qury sellect ke table `order` dengan penambahan setup query
  - solusi bisa ambil data dari replika, tapi tetap digunakan
