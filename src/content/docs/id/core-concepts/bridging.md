---
title: Bridging
description: Core concept page for "Bridging".
---

Bridge merupakan fondasi bagi pengguna dan aplikasi lintas rantai. Pengguna mungkin datang ke rantai lain, seperti Taiko (sebuah ZK-rollup). Untuk melakukannya, mereka perlu mengalirkan dana. Secara terkenal, pengaliran dana telah menjadi operasi yang berbahaya. Bagaimana Anda memastikan bahwa Bridge ini aman?

Mari kita jelaskan pengaliran dana di Taiko. Kami akan menjawab pertanyaan-pertanyaan berikut:

- [Bagaimana protokol Taiko memungkinkan pesan lintas-rantai yang aman?](#pesan-lintas-rantai)
- [Apa itu layanan sinyal Taiko?](#layanan-sinyal)
- [Bagaimana implementasi Bridge Taiko bekerja?](#bagaimana-Bridge-bekerja)

## Pesan Lintas-rantai

Desain protokol Taiko, khususnya kesetaraan Ethereum-nya, memungkinkan pesan lintas-rantai yang aman. Mari kita lihat bagaimana cara kerjanya dengan menggunakan bukti merkle yang sederhana.

### Taiko menyimpan hash blok dari setiap rantai

Taiko menyebarkan dua kontrak pintar yang menyimpan hash dari rantai lain:

- TaikoL1 menyimpan pemetaan `l2Hashes` berdasarkan blockNumber->blockHash (disebarkan di Ethereum)
- TaikoL2 menyimpan pemetaan `l1Hashes` berdasarkan blockNumber->blockHash (disebarkan di Taiko)

Setiap kali blok L2 dibuat di Taiko, hash blok yang melingkupinya di L1 disimpan dalam kontrak TaikoL2. Dan setiap kali blok L1 diverifikasi, hash L2 disimpan dalam kontrak TaikoL1 (hanya yang terbaru, jika beberapa diantaranya diverifikasi sekaligus).

### Tree Merkle memungkinkan memverifikasi nilai ada di rantai lain

Tree Merkle adalah struktur penyimpanan data yang memungkinkan banyak data di-"fingerprint" dengan satu hash tunggal, yang disebut root merkle. Cara struktur ini memungkinkan seseorang memverifikasi bahwa suatu nilai ada dalam struktur data besar ini, tanpa benar-benar perlu memiliki akses ke seluruh Tree merkle. Untuk melakukannya, verifier akan membutuhkan:

- root merkle, ini adalah hash "fingerprint" tunggal dari Tree merkle
- Nilai, ini adalah nilai yang kita periksa ada di dalam root merkle
- Daftar hash saudara perantara, ini adalah hash yang memungkinkan verifier untuk menghitung kembali root merkle

Anda dapat mendapatkan root merkle terbaru yang diketahui disimpan di rantai tujuan dengan memanggil `getCrossChainBlockHash(0)` pada kontrak TaikoL1/TaikoL2. Anda dapat mendapatkan nilai/pesan untuk diverifikasi dan hash saudara untuk root merkle terbaru tersebut dengan meminta dengan panggilan RPC standar `eth_getProof` pada "rantai sumber". Kemudian Anda hanya perlu mengirimnya untuk diverifikasi terhadap hash blok terbaru yang diketahui disimpan dalam daftar di "rantai tujuan".

Seorang verifier akan mengambil nilai (sehelai daun di Tree merkle) dan hash saudara untuk menghitung kembali root merkle. Jika root merkle yang dihitung cocok dengan yang disimpan dalam daftar hash blok rantai tujuan (hash blok rantai sumber), maka kita telah membuktikan bahwa pesan itu dikirim di rantai sumber, dengan asumsi hash blok rantai sumber yang disimpan di rantai tujuan benar.

## Layanan Sinyal

Layanan sinyal Taiko adalah kontrak pintar yang tersedia di kedua L1 dan L2, tersedia untuk pengembang dapp mana pun untuk digunakan. Ini adalah yang menggunakan bukti merkle yang telah kita jelaskan sebelumnya untuk menyediakan layanan pesan lintas-rantai yang aman.

Anda dapat menyimpan sinyal dan memeriksa apakah suatu sinyal dikirim dari suatu alamat. Ini juga mengekspos satu fungsi penting lainnya: `isSignalReceived`.

Apa yang dilakukan fungsi ini? Hal pertama yang perlu dipahami adalah bahwa protokol Taiko memelihara dua kontrak penting:

- `TaikoL1`
- `TaikoL2`

Kontrak-kontrak ini sama-sama melacak hash blok pada **rantai lain**. Jadi TaikoL1, yang didistribusikan di Ethereum, memiliki akses ke hash blok terbaru di Taiko. Dan TaikoL2, yang didistribusikan di Taiko, memiliki akses ke hash blok terbaru di Ethereum.

Jadi, `isSignalReceived` dapat membuktikan di salah satu rantai bahwa Anda telah mengirim sinyal ke Layanan Sinyal di rantai lain. Pengguna atau dapp dapat memanggil `eth_getProof`(https://eips.ethereum.org/EIPS/eip-1186) yang menghasilkan bukti merkle.

Anda perlu menyediakan `eth_getProof` dengan:

1. Sinyal (data yang ingin Anda buktikan ada dalam root penyimpanan dari beberapa blok pada rantai)
2. Alamat layanan sinyal (alamat kontrak yang menyimpan sinyal yang diberikan)
3. Nomor blok di mana Anda menegaskan sinyal tersebut dikirimkan (opsional—jika Anda tidak menyediakan ini, akan beralih ke nomor blok terbaru)

Dan, `eth_getProof` akan menghasilkan bukti merkle (akan memberikan hash saudara yang diperlukan dan tinggi blok, yang bersama dengan sinyal, dapat membangun kembali root penyimpanan merkle dari blok di mana Anda menegaskan sinyal ada di dalamnya).

Ini berarti, dengan asumsi bahwa hash yang dipelihara oleh TaikoL1 dan TaikoL2 benar, kita dapat dengan andal mengirimkan **pesan lintas-rantai**.

Mari kita buat contoh:

1. Pertama, kita dapat mengirim pesan pada suatu rantai sumber, dan menyimpannya pada layanan sinyal.
2. Selanjutnya, kita panggil `eth_getProof`, yang akan memberikan bukti bahwa kita memang mengirim pesan pada rantai sumber.
3. Akhirnya, kita panggil `isSignalReceived` pada LayananSinyal di rantai tujuan yang pada dasarnya hanya memverifikasi bukti merkle. `isSignalReceived` akan mencari hash blok yang Anda klaim Anda menyimpan pesan di rantai sumber (tempat Anda awalnya mengirim pesan), dan dengan hash saudara di dalam bukti merkle itu akan membangun kembali root merkle, yang memverifikasi bahwa sinyal tersebut termasuk dalam root merkle tersebut—artinya itu telah dikirim.

Dan voilà! Kita telah mengirimkan pesan lintas-rantai. Jika ini membingungkan, Anda juga dapat menemukan aplikasi dApp sederhana yang dibangun selama salah satu lokrootya kami untuk menunjukkan dasar-dasarnya. Anda dapat menemukannya [di sini](https://github.com/taikoxyz/MessageServiceShowCaseApp).

## Bagaimana Bridge Bekerja

Bridge adalah seperangkat kontrak pintar dan aplikasi web frontend yang memungkinkan Anda mengirimkan ETH dan token ERC-20, ERC-1155, dan ERC-721 testnet antara Ethereum dan Taiko. Bridge ini hanyalah salah satu implementasi yang mungkin dibangun di atas protokol inti Taiko, khususnya layanan sinyal yang dapat digunakan siapa pun untuk membangun Bridge.

Pertama, ini adalah bagan alir cara kerja implementasi aplikasi Bridge kami, yang menggunakan layanan sinyal:

![alur pesan pengiriman Bridge](~/assets/content/docs/core-concepts/bridging-send-message.excalidraw.png) \
![alur pesan proses Bridge](~/assets/content/docs/core-concepts/bridging-process-message.excalidraw.png)

### Bagaimana Bridge Ether Bekerja?

Bridge Taiko menggunakan Layanan Sinyal yang telah kita jelaskan. Berikut adalah alur pengguna umum untuk Bridge Taiko:

1. Pengguna mengirimkan dana mereka ke kontrak Bridge
2. Bridge mengunci Ether, dan menyimpan pesan dengan memanggil `sendSignal(message)` pada kontrak LayananSinyal
3. Pengguna menerima Ether di rantai tujuan, jika mereka (atau yang lain) memberikan bukti merkle yang valid bahwa pesan tersebut diterima di rantai sumber

Dengan desain saat ini ada 2 cara untuk mengalirkan `Ether`:

1. Kasus hanya `Ether`: Pengguna berinteraksi langsung dengan kontrak Bridge dengan memanggil `sendMessage`
2. Kasus `ERC-XXX` + `Ether` +: Pengguna berinteraksi dengan `ERCXXXVault` (ERC20, ERC721, ERC1155) karena ingin mengalirkan beberapa token, tetapi jika dia mengisi bidang `message.value`, juga `Ether` akan dikirimkan

### Bagaimana Bridge ERC-20 (atau ERC-721, ERC-1155) Bekerja?

Token ERC-20 berasal dari suatu rantai kanonikal. Untuk mengirim token dan mengalirkannya ke rantai lain, kontrak BridgedERC20 baru perlu didistribusikan di rantai tujuan.

#### Bridge dari rantai kanonikal ke rantai tujuan

Berikut langkah-langkah umum untuk mentransfer ERC-20 kanonikal (prosesnya sama untuk jenis token ERC-721, dan ERC-1155 juga!) dari rantai sumber ke rantai tujuan:

1. Kontrak untuk ERC-20 (atau ERC-721, ERC-1155) harus pertama kali didistribusikan di rantai tujuan (akan dilakukan secara otomatis oleh ERC20Vault jika belum didistribusikan)

2. Panggil `sendToken` pada ERC20Vault rantai sumber, ini akan **mentransfer** jumlahnya dengan menggunakan fungsi `safeTransferFrom` pada kontrak ERC-20 kanonikal, di rantai sumber, ke ERC20Vault.

3. Kontrak vault (melalui Bridge) mengirim pesan ke Layanan Sinyal (pada rantai sumber), pesan ini akan berisi beberapa metadata terkait permintaan Bridge, tetapi yang paling penting termasuk calldata untuk metode `receiveToken`.

4. Proses pesan di rantai tujuan dengan mengirimkan bukti merkle (dihasilkan dari rantai sumber), membuktikan bahwa pesan termasuk dalam keadaan Layanan Sinyal rantai sumber. Setelah memverifikasi ini terjadi dan melakukan beberapa pemeriksaan, akan mencoba untuk memanggil metode `receiveToken` yang dienkripsi dalam pesan. Ini akan **membuat** ERC-20 (atau ERC-721, ERC-1155) pada kontrak BridgedERC20 ke alamat `to` di rantai tujuan!

#### Bridge dari rantai tujuan kembali ke rantai kanonikal

Oke sekarang mari kita lakukan sebaliknya, bagaimana kita mentransfer token yang diBridgei dari rantai sumber ke rantai tujuan? (Rantai tujuan dalam hal ini adalah rantai kanonikal, di mana token asli berada.)

1. Kontrak untuk ERC-20 (atau ERC-721, ERC-1155) sudah ada di rantai kanonikal, jadi tidak perlu mendistribusikan yang baru.

2. Panggil `sendToken` pada kontrak vault token rantai sumber, ini akan **membroot** ERC-20 pada kontrak BridgedERC20.

3. Kontrak vault (melalui Bridge) mengirim pesan ke Layanan Sinyal (pada rantai sumber), pesan ini akan berisi beberapa metadata terkait permintaan Bridge, tetapi yang paling penting termasuk calldata untuk metode `receiveToken`.

4. Proses pesan di rantai tujuan dengan mengirimkan bukti merkle (dihasilkan dari rantai sumber), membuktikan bahwa pesan termasuk dalam keadaan Layanan Sinyal rantai sumber. Setelah memverifikasi ini terjadi dan melakukan beberapa pemeriksaan, akan mencoba untuk memanggil metode `receiveToken` yang dienkripsi dalam pesan ini. Ini akan **mentransfer** jumlah dari Vault Token di rantai tujuan ke alamat `to` di rantai tujuan.
