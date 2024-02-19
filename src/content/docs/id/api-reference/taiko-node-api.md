---
title: Taiko Node API
description: Taiko Node API mendeskripsikan berbagai permukaan API dari sebuah node Taiko.
---

Menggunakan node Taiko seharusnya terasa sama seperti menggunakan node L1 lainnya, karena pada dasarnya kami menggunakan klien L1 dan melakukan beberapa modifikasi yang kompatibel ke belakang. Anda dapat membaca tentang arsitektur node Taiko [di sini](/id/core-concepts/taiko-nodes).

## Perbedaan dari klien Geth

Lihat halaman perbedaan fork untuk melihat seperangkat perubahan minimal yang dilakukan pada Geth di [sini](https://geth.taiko.xyz).

## Eksekusi API JSON-RPC

Periksa spesifikasi klien eksekusi [di sini](https://ethereum.github.io/execution-apis/api-documentation/).

## Engine API

Periksa spesifikasi API mesin [di sini](https://github.com/ethereum/execution-apis/blob/main/src/engine/common.md).

## Harness uji Hive

Jika sebuah node Taiko seharusnya terasa sama seperti menggunakan node L1 lainnya, maka tentu saja seharusnya dapat lulus uji harness e2e hive. Pada saat penulisan ini, tes hive sebenarnya merupakan salah satu referensi terbaik untuk apa yang sebenarnya API dari sebuah node Ethereum.

Kami sedang bekerja untuk mengintegrasikan dengan hive, jadi tetap terhubung!