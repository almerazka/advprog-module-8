**Tutorial 8 Pemrograman Lanjut (Advanced Programming) 2024/2025 Genap**
* Nama    : Muhammad Almerazka Yocendra
* NPM     : 2306241745
* Kelas   : Pemrograman Lanjut - A

# Reflection High Level Networking
**🍊 1. What are the key differences between unary, server streaming, and bi-directional streaming RPC (_Remote Procedure Call_) methods, and in what scenarios would each be most suitable?**
| Jenis RPC                    | Cara Kerja                                                      | Kelebihan                                                                  | Kekurangan                                                              | Contoh Penggunaan                                                        | Kapan Digunakan                                                       |
|------------------------------|------------------------------------------------------------------|----------------------------------------------------------------------------|-------------------------------------------------------------------------|---------------------------------------------------------------------------|-----------------------------------------------------------------------|
| **Unary RPC**                     | Client mengirim satu _request_ dan menerima satu _response_ dari server.     | Paling sederhana dan mudah diimplementasikan.                               | Tidak cocok untuk pengiriman data besar atau berkala.                   | Login pengguna, Mengambil data user, Memvalidasi form                     | Jika hanya butuh satu permintaan dan satu jawaban.                               |
| **Server Streaming RPC**          | Client mengirim satu _request_, lalu server mengirim banyak _response_.      | Efisien untuk mengirim data bertahap atau data berkelanjutan.               | Client tidak bisa kirim data lagi setelah _request_ awal.               | Streaming berita, Data cuaca _real-time_, Harga saham _live_              | Jika server harus mengirim banyak data atau _update_ terus-menerus.              |
| **Client Streaming RPC**          | Client mengirim banyak _request_, server memberikan satu _response_ akhir.   | Memungkinkan client untuk terus mengirim data bertahap dalam satu koneksi.  | Server harus menunggu semua data masuk sebelum bisa merespons.          | Upload file besar, Mengirim batch data sensor, Form multi-step            | Jika client perlu kirim data banyak dan bertahap, lalu tunggu hasilnya.          |
| **Bi-directional Streaming RPC**  | Client dan server saling kirim banyak pesan secara berurutan atau bersamaan. | Komunikasi dua arah secara _real-time_ dan sangat fleksibel.                | Paling kompleks untuk dikelola dan disinkronkan.                        | Aplikasi _chat_, _Video call_, _Multiplayer game_                         | Jika membutuhkan komunikasi dua arah secara _real-time_ atau data terus-menerus. |

---
**🍋 2. What are the potential security considerations involved in implementing a gRPC service in Rust, particularly regarding authentication, authorization, and data encryption?**
- **Autentikasi** : Pertama-tama pastikan bahwa pengguna atau sistem yang mengakses layanan gRPC adalah benar dan sah.
  1. Gunakan TLS agar _client_ dan server saling memverifikasi identitas melalui sertifikat digital.
  2. Tambahkan token autentikasi, seperti **JWT** (JSON Web Token), dalam metadata header di setiap permintaan **gRPC**.
  3. Gunakan _middleware_ (misalnya dengan _tonic_ + _tower_) untuk mengecek validitas token pada awal setiap permintaan.
  4. Periksa juga apakah token masih berlaku, belum kedaluwarsa, dan ditandatangani dengan benar.
  
- **Otorisasi** : Setelah identitas diverifikasi, otorisasi dilakukan untuk memastikan pengguna hanya bisa mengakses layanan yang sesuai dengan haknya. Kita ngecek apakah pengguna memiliki izin untuk mengakses layanan tertentu. 
  1. Terapkan **Role-Based Access Control** (RBAC): atur hak akses berdasarkan peran seperti _admin_, _user_, dll.
  2. Gunakan _interceptor_ atau _middleware_ untuk mengecek _role_ pengguna berdasarkan klaim di token **JWT**.
  3. Cek metadata seperti _permissions_, _scopes_, atau _role_ sebelum mengeksekusi metode RPC tertentu.

- **Enkripsi Data** : Untuk mencegah pencurian atau manipulasi data selama proses komunikasi, data harus dienkripsi saat dikirim dan (jika perlu) disimpan.
  1. Aktifkan **TLS** (_Transport Layer Security_) di server dan client **gRPC** untuk mengenkripsi data dalam perjalanan.
  2. Gunakan sertifikat **SSL** yang valid dan hindari penggunaan 'insecure_channel', terutama di lingkungan produksi.
  3. Untuk data penting yang disimpan, pertimbangkan juga enkripsi saat disimpan.
  4. Pastikan komunikasi hanya dilakukan lewat _channel_ aman (**HTTPS/TLS**).
 
---
**🥑 3. What are the potential challenges or issues that may arise when handling bidirectional streaming in Rust gRPC, especially in scenarios like chat applications?**
  1. Sinkronisasi dan Urutan Pesan

     Karena _server_ dan _client_ bisa saling kirim pesan kapan aja, kita harus pastiin pesan-pesan itu gak ketuker urutannya atau bahkan hilang. Solusinya gunakan channel _asynchronous_ seperti 'tokio::sync::mpsc', dan implementasikan sistem penomoran pesan atau _timestamp_ biar gampang diatur 
  
  2. Manajemen Koneksi dan Deteksi Kesalahan

     Koneksi _streaming_ yang terus-menerus rentan terhadap putus koneksi, _client_ tidak responsif, atau _timeout_ jaringan. Hal ini penting untuk ditangani agar server tidak terus menunggu pesan dari _client_ yang sudah terputus. Makanya perlu dicek terus koneksinya aktif atau nggak (pakai _heartbeat_, _timeout_, atau _retry_ berbasis `tokio::time::timeout`.), dan kalau putus, bisa nyambung lagi dengan aman
  
  3. Skalabilitas dan Konsumsi Resource

     Aplikasi chat berskala besar bisa memiliki ratusan atau ribuan klien yang terhubung secara bersamaan, nah server harus kuat menangani semua _streaming_ itu tanpa jadi lambat. Jadi penting banget pakai sistem yang hemat memori dan nggak bikin CPU kerja keras terus. Rust + _async_ (pakai _tokio_) bantu banget buat hal ini.

  4. Keamanan dan Validasi Data

     Setiap pesan yang dikirim oleh _client_ harus divalidasi agar tidak mengandung skrip berbahaya (XSS atau injeksi data) dan digunakan untuk spam atau serangan **DoS**. Makanya, harus ada _middleware_ validasi sebelum pesan diproses, pembatasan ukuran dan frekuensi pesan, dan serta otentikasi tiap klien.
     
  5. Race Condition dan Deadlock

     Karena semuanya jalan bareng-bareng (_asynchronous_), kadang dua hal bisa kejadian bersamaan dan bentrok. Ini bisa bikin _error_ aneh atau malah aplikasi macet. Solusinya manfaatkan `tokio::select!` untuk menangani banyak task secara aman, dan pastikan tidak ada _dependency_ siklik antar _resource_.

---   
**🫐 4. What are the advantages and disadvantages of using the `tokio_stream::wrappers::ReceiverStream` for streaming responses in Rust gRPC services?**

**Kelebihan**

  - **Mendukung Asynchronous secara Native** : **ReceiverStream** dibangun di atas `tokio::sync::mpsc::Receiver`, yang sepenuhnya _asynchronous_ dan berjalan optimal di dalam ekosistem Tokio. Hal ini memungkinkan pengiriman data antar task secara _non-blocking_, menjaga performa aplikasi tetap responsif dan efisien dalam konteks **gRPC** _streaming_.
  
  - **Integrasi Mudah dengan Pola Produser-Konsumer** : Cocok untuk arsitektur di mana satu _task_ menghasilkan data (produser) dan _task_ lain mengkonsumsinya secara bertahap (konsumer), seperti aplikasi chat, notifikasi _real-time_, atau server push pada **gRPC**.

  - **Penggunaan Sederhana** : **ReceiverStream** menyederhanakan konversi dari _channel_ ke _stream_, menjadikannya praktis untuk digunakan dalam implementasi _server-side streaming_ atau _bidirectional streaming_ di **gRPC**.

  - **Kompatibilitas Tinggi dalam Ekosistem Tokio** : Terintegrasi erat dengan fitur Tokio lainnya seperti `select!`, _timeout_, _cancellation_, dan _task asynchronous_, menjadikannya fleksibel dan _powerful_ untuk penanganan aliran data di server **gRPC** berbasis _async_.

**Kekurangan**

  - **Kurang Optimal untuk _Throughput_ Tinggi** : Pada skenario lalu lintas data besar dan cepat, **ReceiverStream** dapat mengalami _bottleneck_ karena _buffer_ yang penuh. Hal ini bisa menyebabkan _latency_ meningkat atau bahkan kehilangan pesan jika tidak dikonfigurasi dengan baik.

  - **Pengelolaan Error dan Flow Control Manual** : Tidak menyediakan mekanisme bawaan untuk menangani kesalahan atau _flow control_. Pengembang harus menangani kondisi seperti koneksi terputus, _error parsing_, atau pemrosesan lambat secara eksplisit dalam kode **gRPC**-nya.

  - **Potensi Overhead pada Penggunaan Memori** : Jika konsumen lambat dalam membaca data dari stream, pesan akan menumpuk di _buffer_ `mpsc`. Hal ini bisa mengakibatkan konsumsi memori meningkat drastis, yang berisiko menurunkan performa sistem atau menyebabkan _panic_ jika tidak ditangani.

  - **Kurva Belajar untuk Pemula** : Meskipun _interface_-nya terlihat sederhana, pemahaman mendalam tentang _asynchronous programming_ di **Rust** (terutama dengan `Tokio`, `mpsc`, dan `Stream`) tetap diperlukan. Hal ini bisa menjadi tantangan bagi pengembang yang baru memulai dengan _async_ **Rust** atau **gRPC**.

--- 
**🍎 5. In what ways could the Rust gRPC code be structured to facilitate code reuse and modularity, promoting maintainability and extensibility over time?**

  1. **Pisahkan Definisi gRPC dan Implementasi** : Menggunakan file `.proto` untuk mendefinisikan layanan dan pesan **gRPC**, lalu hasilkan kode **Rust** secara otomatis. Simpan kode hasil generate dan logika implementasi di modul yang berbeda agar lebih terorganisir.
  
  2. **Struktur Proyek Modular** : Membagi kode ke dalam modul seperti _server_, _client_, _service_, _handler_, _config_, dan _utils_. Ini membantu pengembangan per bagian secara terpisah dan memudahkan pengujian.

  3. **Gunakan Trait untuk Abstraksi** : Membuat _trait_ sebagai kontrak dari layanan yang memudahkan pengujian (_mocking_) dan mengganti implementasi tanpa mengubah keseluruhan kode.
  
  4. **Manfaatkan Middleware dan Crate Eksternal** : Menggunakan _middleware_ untuk autentikasi, _logging_, dan _error handling_ agar dapat digunakan ulang di banyak layanan. Gunakan **crate** seperti `tonic`, `tokio`, dan `tracing` untuk meningkatkan efisiensi dan keterbacaan.
  
  5. **Terapkan Prinsip SOLID** : Seperti pemisahan tanggung jawab (_single responsibility_) dan terbuka untuk ekstensi (_open for extension_), agar kode tetap fleksibel terhadap perubahan.

---
**🍍 6. In the MyPaymentService implementation, what additional steps might be necessary to handle more complex payment processing logic?**

  - **Validasi Data** : Pastikan semua data yang diterima dari klien valid dan lengkap. Misalnya, cek apakah jumlah pembayaran tidak negatif, kartu kredit valid, atau saldo cukup.

  - **Autentikasi dan Otorisasi** : Memverifikasi hanya pengguna yang sah dan berhak yang dapat melakukan transaksi untuk mencegah penyalahgunaan layanan.

  - **Integrasi dengan Gateway Pembayaran** : Menghubungkan layanan dengan penyedia pembayaran eksternal (seperti `Midtrans`, `Stripe`, dll) untuk memproses transaksi secara nyata.

  - **Manajemen Error dan Keandalan Transaksi** : Menangani kasus seperti pembayaran gagal atau _timeout_ dengan baik, termasuk _retry_ dan _rollback_ jika perlu.

  - **Keamanan Data** : Lindungi data sensitif seperti nomor kartu atau token transaksi dengan enkripsi dan hindari menyimpan atau mencetak data pribadi ke log untuk melindungi privasi pengguna.

---
**🫛 7. What impact does the adoption of gRPC as a communication protocol have on the overall architecture and design of distributed systems, particularly in terms of interoperability with other technologies and platforms?**

  a. **Interoperabilitas** : **gRPC** memungkinkan layanan yang dikembangkan dengan berbagai bahasa pemrograman (seperti _Rust_, _Java_, dan _JavaScript_) untuk berkomunikasi secara mulus melalui definisi protokol bersama di file `.proto`.
  
  b. **Kecepatan dan Efisiensi** : Dengan `HTTP/2` dan **Protocol Buffers**, **gRPC** lebih cepat dan efisien dibandingkan dengan **JSON** atau **XML**, mengurangi _overhead_ komunikasi dan meningkatkan performa.

  c. **Skalabilitas** : **gRPC** mendukung _streaming_ dan protokol biner, menjadikannya ideal untuk aplikasi dengan kebutuhan skalabilitas tinggi, seperti _microservices_.
  
  d. **Desain yang Terstruktur** : Definisi API yang jelas di `.proto` mempermudah pengelolaan komunikasi antar layanan, memastikan konsistensi dan mengurangi kesalahan.
  
  e. **Keterbatasan HTTP/2** : Meskipun efisien, **gRPC** bergantung pada `HTTP/2`, yang mungkin tidak didukung oleh beberapa infrastruktur atau platform lama.

---
**🍌 8. What are the advantages and disadvantages of using HTTP/2, the underlying protocol for gRPC, compared to HTTP/1.1 or HTTP/1.1 with WebSocket for REST APIs?**

**Keuntungan**

  1. **Multiplexing (Koneksi Lebih Efisien)** : `HTTP/2` mendukung banyak permintaan/respons sekaligus dalam satu koneksi TCP tanpa saling menunggu, sedangkan `HTTP/1.1` hanya bisa mengirim satu permintaan dalam satu waktu per koneksi.
  2. **Kompresi Header** : `HTTP/2` menggunakan kompresi header (HPACK) yang mengurangi ukuran data permintaan, mempercepat komunikasi, dan menghemat bandwidth. `HTTP/1.1` mengirim header secara penuh di setiap permintaan.
  3. **Latency Rendah & Streaming Dua Arah** : Cocok untuk layanan yang membutuhkan komunikasi _real-time_ atau _transfer_ data terus-menerus (_streaming_), seperti yang biasa dilakukan dengan **WebSocket**.
  4. **Binary Protocol** : `HTTP/2` menggunakan format biner, bukan teks seperti `HTTP/1.1`. Ini membuat _parsing_ lebih cepat dan efisien untuk mesin (misalnya, cocok dengan _Protobuf_ di **gRPC**).

**Kekurangan**

  1. **Kompleksitas Implementasi** : Protokol `HTTP/2` (dan **gRPC**) lebih kompleks dibanding `HTTP/1.1` atau **WebSocket**. Diperlukan alat tambahan dan pemahaman mendalam, terutama dalam _debugging_ dan pengujian.
  2. **Keterbatasan Dukungan di Browser** : **gRPC** tidak berjalan langsung di browser karena bergantung pada `HTTP/2` dan _Protobuf_. Berbeda dengan **REST API** berbasis `HTTP/1.1` yang dapat diakses langsung dari browser.
  3. **Overkill untuk Aplikasi Sederhana** : Jika aplikasi hanya memerlukan permintaan data dasar (misalnya: `GET/POST` biasa), maka keunggulan `HTTP/2` tidak akan terlalu terasa, dan `HTTP/1.1` mungkin sudah cukup.
  4. **Masalah Kompatibilitas** : Tidak semua jaringan/perangkat mendukung `HTTP/2` dengan baik (misalnya, _proxy_ tua atau perangkat _edge_ tertentu bisa memblokir atau merusak trafik `HTTP/2`).


---
**🍒 9. How does the request-response model of REST APIs contrast with the bidirectional streaming capabilities of gRPC in terms of real-time communication and responsiveness?**

**REST API dan Model Request-Response**

**REST API** menggunakan model permintaan dan jawaban (_request-response_), artinya 
- Klien harus mengirimkan request terlebih dahulu untuk mendapatkan _response_ dari server.
- Komunikasi ini bersifat satu arah dan sinkron, di mana klien menunggu hingga server selesai memproses permintaan.
- Dalam aplikasi _real-time_, **REST** kurang efisien, karena klien harus melakukan polling berulang kali (mengirim permintaan setiap beberapa detik) hanya untuk mengecek apakah ada data baru.
- **Contoh kasus** : Aplikasi _chat_ dengan **REST** perlu mengirim permintaan setiap beberapa detik untuk mengecek pesan baru. Ini membuatnya tidak efisien dan lambat merespons perubahan data.

**gRPC dan Bidirectional Streaming**

**gRPC** mendukung bidirectional streaming, artinya
- Klien dan server bisa mengirim dan menerima data secara bersamaan dan berkelanjutan dalam satu koneksi terbuka.
- Komunikasi ini bersifat dua arah dan asinkron, tanpa perlu menunggu permintaan atau jawaban sebelumnya selesai.
- Cocok untuk aplikasi yang memerlukan komunikasi _real-time_ dan responsivitas tinggi, seperti _video call_, _monitoring_ sistem, atau aplikasi _chat_.
- **Contoh kasus**: Pada aplikasi _chat_ berbasis **gRPC**, pengguna bisa langsung menerima pesan baru dari server tanpa harus melakukan _polling_, dan bisa mengirim pesan kapan saja, mirip seperti **WhatsApp** atau **Telegram**

---
**🌽 10. What are the implications of the schema-based approach of gRPC, using Protocol Buffers, compared to the more flexible, schema-less nature of JSON in REST API payloads?**

1. **Pendekatan Schema-Based pada gRPC (Protocol Buffers)**
   
   **gRPC** menggunakan _Protocol Buffers_, format biner ringan yang membutuhkan definisi struktur data melalui file `.proto`. Ini disebut pendekatan berbasis skema, karena setiap pesan yang dikirim/diterima harus mengikuti struktur yang sudah didefinisikan sebelumnya, validasi otomatis terjadi saat data tidak sesuai skema, dformat data lebih ringkas dan cepat, cocok untuk komunikasi antar-_microservice_ dalam sistem besar atau aplikasi _real-time_.

   - **Keuntungan** : Konsistensinya tinggi jadi semua pihak (_client-server_) tahu persis format datanya, performa lebih cepat dan efisien karena data berbentuk biner dan sudah terstruktur.
   - **Kekurangan** : Kurang fleksibel, jika ada perubahan skema (misal penambahan _field_), semua layanan terkait harus diperbarui. Membutuhkan _tools_ tambahan (misalnya _protoc_ _compiler_) untuk _generate_ kode dari file `.proto`.
   
2. **Pendekatan Schema-less pada REST (JSON)**

   **REST API** biasanya menggunakan **JSON** yang tidak membutuhkan skema formal. Klien dan server bebas menentukan dan mengubah struktur datanya kapan saja.

   - **Keuntungan** : Fleksibel, format data bisa berubah tanpa harus menyinkronkan semua sistem, mudah dipahami manusia dan di _test_ langsung (bisa dibuka dengan _text editor_ biasa).
   - **Kekurangan** : Rentan inkonsistensi, karena tidak ada skema wajib, bisa terjadi perbedaan struktur antara _client_ dan _server_.  Performa lebih rendah karena **JSON** cenderung lebih besar dan _parsing_-nya lebih lambat dibanding format biner seperti _Protobuf_.
