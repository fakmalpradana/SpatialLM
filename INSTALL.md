# Panduan Instalasi dan Pengujian SpatialLM pada Windows 11

Dokumen ini menjelaskan langkah-langkah teknis instalasi SpatialLM pada lingkungan sistem operasi Windows. Dokumentasi ini disusun berdasarkan proses implementasi dan *troubleshooting* yang telah dilakukan untuk menangani kendala kompatibilitas sistem.

## Environment

*   **Sistem Operasi:** Windows 10/11
*   **Python:** Versi 3.11.9
*   **Akselerasi GPU:** NVIDIA GPU dengan dukungan CUDA 12.4
*   **Manajemen Paket:** Poetry dan Pip

## Langkah Instalasi

### 1. Konfigurasi Environment Python

Pastikan direktori kerja aktif adalah root dari repositori repository dan virtual environment (`venv`) Python telah diaktifkan. Verifikasi versi Python untuk memastikan kompatibilitas:

```powershell
.\venv\Scripts\python.exe --version
# Output yang diharapkan: Python 3.11.9
```

### 2. Instalasi PyTorch dan Dependensi Dasar

Mengingat ukuran paket PyTorch yang besar dan potensi kendala *timeout* jaringan pada `poetry`, instalasi PyTorch disarankan dilakukan secara manual menggunakan `pip` dengan konfigurasi *timeout* yang diperpanjang:

```powershell
.\venv\Scripts\pip.exe install torch==2.4.1 torchvision==0.19.1 torchaudio==2.4.1 --index-url https://download.pytorch.org/whl/cu124 --timeout 1000
```

### 3. Instalasi Dependensi lain

Gunakan `poetry` untuk mengelola dependensi proyek lainnya. Konfigurasikan poetry untuk tidak membuat virtual environment baru, melainkan menggunakan yang sudah aktif:

```powershell
.\venv\Scripts\pip.exe install poetry
.\venv\Scripts\poetry.exe config virtualenvs.create false --local
.\venv\Scripts\poetry.exe install
```

*Catatan: Jika terjadi kegagalan pada instalasi paket tertentu (misalnya `rerun-sdk` atau dependensi NVIDIA), instal paket tersebut secara manual menggunakan `pip`, lalu jalankan kembali perintah `poetry install`.*

### 4. Penanganan Kendala Kompatibilitas Windows

Modul `torchsparse` dan `flash-attn` memerlukan kompilasi ekstensi C++/CUDA yang kompleks. Pada lingkungan Windows, proses ini memiliki tingkat kegagalan yang tinggi dikarenakan perbedaan toolchain kompilasi (MSVC).

Sebagai solusi teknis (*workaround*) agar model tetap dapat beroperasi tanpa modul terkompilasi tersebut, dilakukan modifikasi pada kode sumber untuk menonaktifkan fitur `flash-attn` pada backbone `Sonata`. Ini akan mengalihkan komputasi ke implementasi attention standar PyTorch.

**Modifikasi File:** `spatiallm/model/spatiallm_qwen.py`

Pada kelas inisialisasi `Sonata` (sekitar baris 62), tambahkan argumen `enable_flash=False`:

```python
self.point_backbone = Sonata(
    # ... parameter lainnya ...
    num_bins=point_config["num_bins"],
    enable_flash=False,  # <--- Modifikasi Kritikal: Nonaktifkan flash attention
)
```

Setelah modifikasi, pastikan paket terinstal dalam mode *editable* agar perubahan kode terbaca oleh sistem:

```powershell
.\venv\Scripts\pip.exe install -e .
```

## Prosedur Verifikasi

Langkah berikut digunakan untuk memvalidasi fungsionalitas model paska instalasi.

1.  **Unduh Dataset Pengujian:**
    
    Unduh sampel *point cloud* dari repositori HuggingFace:
    ```powershell
    $env:PYTHONIOENCODING="utf-8"
    .\venv\Scripts\huggingface-cli.exe download manycore-research/SpatialLM-Testset pcd/scene0000_00.ply --repo-type dataset --local-dir .
    ```

2.  **Eksekusi Inferensi Model:**
    
    Jalankan skrip inferensi. Penting untuk menggunakan flag `--inference_dtype float32` guna menjamin stabilitas numerik saat *flash attention* dinonaktifkan. Penggunaan `float16`/`bfloat16` tanpa *flash attention* berpotensi menghasilkan output kosong.

    ```powershell
    .\venv\Scripts\python.exe inference.py --point_cloud pcd/scene0000_00.ply --output scene0000_00.txt --model_path manycore-research/SpatialLM1.1-Qwen-0.5B --inference_dtype float32
    ```

3.  **Validasi Hasil:**
    
    Periksa keberadaan dan konten file output `scene0000_00.txt`. Jika instalasi berhasil, file ini akan berisi deskripsi tekstual dari elemen spasial (dinding, pintu, jendela, bounding box objek).
    
    Untuk visualisasi 3D:
    ```powershell
    .\venv\Scripts\python.exe visualize.py --point_cloud pcd/scene0000_00.ply --layout scene0000_00.txt --save scene0000_00.rrd
    rerun scene0000_00.rrd
    ```

---
**Kesimpulan Teknis:** Instalasi pada Windows memerlukan pendekatan hybrid (instalasi pip manual dan poetry) serta penyesuaian kode sumber untuk melewati ketergantungan pada ekstensi C++ yang tidak kompatibel. Dengan konfigurasi di atas, SpatialLM dapat berjalan dengan fungsionalitas penuh untuk inferensi, meskipun tanpa optimasi kecepatan dari kernel khusus.
