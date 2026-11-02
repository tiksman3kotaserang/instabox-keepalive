# instabox-keepalive

Keepalive GitHub Actions untuk mesin InstaBox `hermes` (m_agqykepoj5i5vrfkw4w2z2exum), dijalankan oleh workflow `.github/workflows/keepalive.yml` setiap 5 menit (`*/5 * * * *`) plus `workflow_dispatch`.

## Pertahanan 1: melawan idle pause

Mesin gratis InstaBox memiliki `autoSuspendAfterSeconds` = 1800 (30 menit). Setiap run menjalankan `instabox exec` ringan (`date -u; uptime; echo keepalive-ok`) yang dihitung sebagai aktivitas, sehingga timer idle direset. Jika mesin sudah tertidur karena idle (`autoSuspendedAt` terisi), exec akan membangunkannya kembali.

## Pertahanan 2: melawan batas max-run 480 menit

Mesin gratis berjalan maksimal 480 menit dalam satu run; setelah itu mesin dijeda dan ada cooldown 5-20 menit. Workflow membaca data mesin lewat API, menghitung usia run bila field waktunya tersedia, dan bila usia >= 450 menit (atau bila mode `rotate` dipilih secara manual) melakukan rotasi: `POST /v1/machines/{id}/suspend`, tunggu ~15 detik, lalu `POST /v1/machines/{id}/resume` dengan header `X-API-Key` dan `Idempotency-Key`. Suspend manual tidak memicu cooldown dan wake berikutnya memulai run 480 menit yang baru, dengan proses/memori dipertahankan oleh suspend.

## Catatan adaptasi

Dokumen API mesin tidak mencantumkan field `runStartedAt`/`remainingMinutes`, jadi workflow mencoba beberapa kandidat field waktu dan, bila tidak ada yang tersedia, melewati rotasi otomatis dengan anggun (tetap menjalankan exec). Mode `rotate` selalu memaksa rotasi terlepas dari usia.

## Konfigurasi

- Secret repositori: `INSTABOX_API_KEY` (API key InstaBox bernama `github-keepalive`, kedaluwarsa ~90 hari).
- Mode `workflow_dispatch`: `check` = exec + laporan saja; `rotate` = exec + paksa rotasi suspend-resume.
