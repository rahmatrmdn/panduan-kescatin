# Kescatin - Aplikasi Skrining Kehamilan

Selamat datang di repositori aplikasi Kescatin. Aplikasi ini digunakan untuk melakukan skrining kesehatan pada Calon Pengantin (Catin) untuk menentukan kelayakan kehamilan berdasarkan faktor-faktor medis.

## Alur Kuesioner Skrining Kehamilan

Aplikasi menentukan status kelayakan hamil menjadi 3 kategori utama:
1. **Hijau**: Kondisi ideal, boleh hamil.
2. **Kuning**: Risiko menengah, boleh hamil dengan pengawasan dokter.
3. **Merah**: Risiko tinggi, tunda kehamilan dan segera perbaiki kesehatan (atau perlu ke dokter spesialis jika sedang hamil).

Berikut adalah alur bagaimana aplikasi menyimpulkan status skrining dari pengisian data fisik dan riwayat kesehatan.

```mermaid
flowchart TD
    Start(["Mulai Pengisian Kuesioner Skrining"]) --> Input["Input Data: Usia, Fisik, Riwayat Obsterik, & Riwayat Penyakit"]
    Input --> EvaluasiRisiko{"Evaluasi Seluruh Faktor Risiko<br>Apakah terdapat kriteria berisiko?"}

    %% Kriteria Risiko Tinggi
    EvaluasiRisiko -->|"Ya"| CekRisikoTinggi{"Apakah Memenuhi<br>Kriteria Risiko Tinggi?"}
    
    noteMerah["Daftar Kriteria Risiko Tinggi:<br>- Usia kurang dari 20 tahun<br>- Usia lebih dari 49 tahun<br>- Jarak antar kehamilan kurang dari 2 tahun<br>- Jumlah anak lebih dari 2<br>- IMT kurang dari 18,5 (Kurus)<br>- Lingkar Lengan Atas (LILA) kurang dari 23,5 cm (Kurang Energi Kronis)<br>- Kadar Hb kurang dari 12 g/dL (Anemia) atau Tidak Tahu<br>- Menjawab 'Ya' atau 'Tidak Tahu' pada Riwayat Penyakit (Hipertensi, Diabetes, Asma, Jantung, Ginjal, dll)<br>- Menjawab 'Ya' atau 'Tidak Tahu' pada Infeksi (TB Paru, Malaria, Sifilis, HIV, Hepatitis B, TORCH)<br>- Calon Ibu adalah Penderita Talasemia"]
    CekRisikoTinggi -.- noteMerah

    %% Kriteria Risiko Menengah
    CekRisikoTinggi -->|"Tidak<br>(Lanjut Cek Menengah)"| CekRisikoMenengah{"Apakah Memenuhi<br>Kriteria Risiko Menengah?"}
    
    noteKuning["Daftar Kriteria Risiko Menengah:<br>- Usia 36 hingga 49 tahun<br>- Tinggi badan kurang dari 145 cm<br>- IMT lebih dari 25 (Kelebihan berat badan / Obesitas)<br>- Memiliki riwayat persalinan yang buruk<br>- Calon Ibu Pembawa & Calon Ayah Pembawa/Penderita Talasemia<br>- Salah satu (Ibu/Ayah) Pembawa/Penderita Hemofilia"]
    CekRisikoMenengah -.- noteKuning

    %% Tidak Ada Risiko
    EvaluasiRisiko -->|"Tidak Ada Sama Sekali"| Hijau["🟢 Kategori HIJAU<br>Kondisi Ideal, Boleh Hamil"]
    CekRisikoMenengah -->|"Tidak<br>(Semua kriteria aman)"| Hijau

    %% Alur untuk Merah
    CekRisikoTinggi -->|"Ya (Minimal 1 Kriteria)"| Merah["🔴 Kategori MERAH<br><br>Jika Belum Hamil: Tunda kehamilan & perbaiki kesehatan<br>Jika Sedang Hamil: Risiko Tinggi, segera ke dokter spesialis"]

    %% Alur untuk Kuning
    CekRisikoMenengah -->|"Ya (Minimal 1 Kriteria)"| Kuning["🟡 Kategori KUNING<br><br>Jika Belum Hamil: Boleh hamil dengan pengawasan ketat tenaga medis<br>Jika Sedang Hamil: Perlu pengawasan ekstra dari nakes"]

    Hijau --> Selesai(["Selesai Skrining"])
    Merah --> Selesai
    Kuning --> Selesai
```
