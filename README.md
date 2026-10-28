# Skematik Sensor RPM & PID (Close Loop Motor DC)

Dokumentasi interaktif rangkaian **close loop kontrol kecepatan motor DC** yang dibangun di atas papan bolong *Mascot PCB002*: open loop (PWM generator), sensor RPM optocoupler, dan kontroler PID analog.

**Demo:** https://fale2323.github.io/skematik-sensor-pid/

## Isi halaman

| Tab | Isi |
|---|---|
| Layout Sensor | Layout PCB bolong per tahap (tampak komponen dan tampak solder), daftar lubang, jalur solder, jumper, dan uji bunyi multimeter |
| Close Loop Final | Diagram blok, skematik lengkap, titik ukur, daftar bahan, urutan menghidupkan, tuning PID, dan simulasi respon |
| Skematik Sensor | EE-SX112 + LM2917N: optocoupler, referensi TACH−, charge pump, output PV |
| Sambungan ke Open Loop | Tabel kabel antar papan dan sambungan modul L298N |
| Skematik PID | Error amplifier, P, I, D, dan penjumlah (2× LM324) |
| Model 3D | Model 3D papan, sensor, disk encoder, motor, driver, dan kabel (three.js) |

## Ringkasan sistem

```
SP (VR15) --> [Σ error] --> PID --> u --> REF komparator --> PWM 50 Hz --> L298N --> Motor DC
                 ^                                                                       |
                 |                                                                  disk encoder
                 +------ PV 0-5 V <-- LM2917N (f ke V) <-- EE-SX112 <-------------------+
```

- **Open loop:** NE555 (sawtooth 50 Hz) → buffer → clamper → penguat → komparator LM324 → zener 4V7 → ENA L298N.
- **Sensor:** optocoupler slot EE-SX112 menghasilkan pulsa 0–9 V, lalu LM2917N mengubah frekuensi menjadi tegangan PV (0–5 V).
- **PID:** `u = Kp·e + Ki∫e dt + Kd·de/dt`, dengan `e = SP − PV`. Memakai 2× LM324 dan suplai ±9 V (−9 V dari baterai 9 V).
- **Penutup loop:** `u` masuk ke J4 papan open loop dan menggantikan VR4 sebagai REF komparator. Jumper JP1 dipindah ke posisi PID (F14–F15).

Setelan awal PID hasil simulasi: VR11 = 15 kΩ, VR12 = 47 kΩ, VR13 = 5 kΩ, dengan C31 = 4,7 µF.

## Komponen utama

NE555, LM324 (×3), LM2917N, optocoupler EE-SX112, modul L298N, motor DC 12 V, disk encoder, trimpot multiturn 3296W, dan potensio panel untuk setpoint.

## Cara membuka

Buka `index.html` di browser. Semuanya satu file HTML, tanpa build.

- Tab **Model 3D** memuat three.js dari CDN, jadi butuh koneksi internet saat pertama dibuka.
- Font memakai Google Fonts. Tanpa internet, tampilan otomatis memakai font bawaan sistem.

## Struktur

```
skematik-sensor-pid/
├── index.html
└── README.md
```

## Penulis

Dzakwan Naufal ([@fale2323](https://github.com/fale2323)), D4 Teknik Komputer, PENS.
