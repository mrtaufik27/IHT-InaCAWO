Catatan Hari 1: In House Training InaCAWO Hindcast 2026
Panduan langkah demi langkah untuk konfigurasi environment, kompilasi model, dan pre-processing data input pada sistem HPC.

1. Koneksi ke HPC DataCenter
Sebelum memulai instalasi dan simulasi, pastikan koneksi SSH ke HPC telah terkonfigurasi. Compiler yang digunakan adalah Intel Compiler.

Generate SSH Key
Jalankan perintah berikut pada terminal lokal untuk membuat pasangan kunci public/private (ed25519):

Bash
ssh-keygen -t ed25519 -C "mahisaajy@gmail.com"
Ikuti instruksi di layar (tekan Enter untuk menyimpan di path default dan lewati passphrase jika tidak diperlukan).

Kunci public (id_ed25519.pub) akan di-generate beserta gambar randomart SHA256.

Login SSH ke HPC
Gunakan username dan IP yang telah diberikan (contoh menggunakan cawohdcst_t1 pada 10.202.26.133):

Bash
ssh cawohdcst_t1@10.202.26.133
(Opsional) Jika ingin menyebutkan spesifik kunci yang digunakan:

Bash
ssh -i /path/to/ssh_key_id_ed25519 cawohdcst_t1@10.202.26.133
Pilih yes untuk menambahkan host ke dalam known_hosts.

2. Kloning Repositori GitHub
Unduh semua repositori yang dibutuhkan untuk dependencies, source code, dan skrip workflow IHT:

Bash
git clone https://github.com/devmetmar/inacawo-deps.git
git clone https://github.com/devmetmar/inacawo-src.git
git clone https://github.com/devmetmar/inacawo-iht.git
3. Instalasi Dependensi (Environment)
Masuk ke direktori dependensi dan jalankan skrip instalasi dalam mode offline (memanfaatkan arsip cache HPC):

Bash
cd inacawo-deps
./install_hindcast_env.bash --offline
Tunggu hingga proses selesai dan terminal menampilkan pesan konfirmasi (==> Done).

4. Kompilasi Model Dasar (InaCAWO)
Skrip kompilasi akan menyiapkan model untuk dua skenario.

Masuk ke direktori source:

Bash
cd inacawo-src
./build_coawst.bash --sbatch
Mengecek Status Kompilasi:
Gunakan perintah squeue untuk memantau status job Anda di sistem scheduler Slurm:

Bash
squeue -u cawohdcst_t7
(Ganti cawohdcst_t7 dengan username Anda)

Memeriksa Hasil Kompilasi:
Jika proses berhasil, akan terbentuk 2 file binari di dalam folder binaries dan file log di folder logs:

Bash
ls binaries/
# Output yang diharapkan: no_ramp_tides  ramp_tides

ls logs/
# Output yang diharapkan: build_no_ramp_tides_...log, compile_cawo_no_ramp...out, dll.
Keterangan Skenario:

ramp_tides: Konfigurasi dengan pengikatan/penyesuaian dinamis (digunakan untuk hari pertama/spin-up).

no_ramp_tides: Konfigurasi standar tanpa penyesuaian awal pasang surut.

5. Menjalankan Pre-processing Data Input (Hindcast)
Proses pre-processing dilakukan di dalam direktori inacawo-iht. Disarankan menggunakan sesi Tmux agar proses running tidak terputus jika koneksi SSH terputus.

Persiapan Environment IHT
Bash
cd inacawo-iht

# Membuat sesi Tmux baru (opsional namun sangat disarankan)
tmux new -s hindcast

# Menyiapkan variabel environment
source ./setup_env.bash
(Tips Tmux: Untuk detach/keluar dari layar Tmux tanpa mematikan proses, tekan Ctrl+B lalu d. Untuk masuk kembali, gunakan perintah tmux attach -t hindcast).

Tahap 1: Mengunduh Data Inisialisasi
Jalankan skrip pre-process untuk mengunduh data sesuai rentang waktu hindcast:

Bash
./run_preprocess.bash --start 20210404 --end 20210410 --stage download --offline
Data yang berhasil disiapkan akan tersimpan secara otomatis pada path scratch:
/scratch/cawohdcst_t7/inacawo-iht/cawo_input/

Tahap 2: Menjalankan Komponen Atmosfer (WPS & WRF)
Eksekusi pre-processing untuk modul WPS (proses ungrib.exe, metgrid.exe, dan real.exe):

Bash
./run_preprocess.bash --start 20210404 --end 20210410 --stage wps
Pantau job yang berjalan di Slurm:

Bash
squeue -u cawohdcst_t1
(Proses akan menampilkan antrean job run_ungr dan run_metg yang terdistribusi di beberapa node seperti mdc370, mdc145, dst).

Tahap 3: Menjalankan Komponen Gelombang (SWAN)
Mempersiapkan data batas boundary dan inisialisasi gelombang laut:

Bash
./run_preprocess.bash --start 20210404 --end 20210410 --stage swan
Tahap 4: Menjalankan Komponen Arus Laut (ROMS)
Mempersiapkan penyesuaian dinamis siklus batimetri/pasang surut, khusus ditandai dengan flag --day1 karena ini merupakan hari inisialisasi:

Bash
./run_preprocess.bash --start 20210404 --end 20210410 --stage roms --day1
