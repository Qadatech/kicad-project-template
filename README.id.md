# PROJECT_NAME

[English](README.md) | **Bahasa Indonesia**

Project hardware KiCad 10.

<!-- template:start -->
## Menggunakan template ini

1. Klik **Use this template → Create a new repository** di GitHub. Nama repository
   akan menjadi nama project KiCad (mis. `sensor-board`).
2. Workflow **Template init** berjalan otomatis pada push pertama:
   - `PROJECT_NAME` diganti dengan nama repository, di nama file maupun isi file
     (`sources/sensor-board/sensor-board.kicad_pro`, `libraries/sensor-board.kicad_sym`, dst.)
   - UUID root schematic di-generate ulang
   - bagian ini dihapus dari kedua README, lalu hasilnya di-commit oleh `github-actions[bot]`
   - workflow **KiCad CI** dijalankan untuk project yang sudah di-rename
3. `git pull`, lalu buka file `.kicad_pro` di KiCad.

Clone lokal tanpa GitHub? Jalankan sendiri:

```bash
scripts/init.sh nama-project   # tanpa argumen: pakai nama folder repository
```

> Jika push dari workflow ditolak, buka **Settings → Actions → General → Workflow permissions**
> dan pilih **Read and write permissions**, lalu jalankan ulang workflow *Template init*
> (tab Actions → Template init → Run workflow).

Setelah inisialisasi, `scripts/init.sh` dan `.github/workflows/template-init.yml` tidak
melakukan apa-apa lagi dan boleh dihapus.
<!-- template:end -->

## Struktur

```
sources/PROJECT_NAME/        KiCad project (.kicad_pro/.kicad_sch/.kicad_pcb) + lib tables
libraries/
  PROJECT_NAME.kicad_sym     simbol khusus project
  PROJECT_NAME.pretty/       footprint khusus project
  PROJECT_NAME.3dshapes/     model 3D (STEP/WRL)
.github/workflows/kicad.yml  ERC, DRC, dan output fabrikasi
```

Library project sudah terdaftar di `sym-lib-table` / `fp-lib-table` project dengan path
`${KIPRJMOD}/../../libraries/...`, jadi tetap jalan setelah di-clone di mana pun.
Untuk model 3D, isi path footprint dengan `${KIPRJMOD}/../../libraries/PROJECT_NAME.3dshapes/<file>.step`.

Title block memakai text variable `${PROJECT}`, `${REVISION}` dan `${CURRENT_DATE}`.
`REVISION` bernilai `dev` di KiCad dan diisi otomatis oleh CI dari tag git / commit.

## CI & rilis

Setiap push / pull request menjalankan [`kicad.yml`](.github/workflows/kicad.yml) di
container `kicad/kicad:10.0`:

| Langkah | Output |
| --- | --- |
| ERC, DRC (+ schematic parity) | `reports/erc.rpt`, `reports/drc.rpt` — job gagal jika ada error |
| Schematic | `PROJECT_NAME-schematic.pdf`, `PROJECT_NAME-bom.csv` |
| PCB | `gerbers/` + `PROJECT_NAME-gerbers.zip`, drill + drill map, `PROJECT_NAME-pos.csv`, `PROJECT_NAME-pcb.pdf`, `PROJECT_NAME.step` |

Output bisa diunduh dari tab **Actions** (artifact). Layer Gerber mengikuti pengaturan
**File → Plot** yang tersimpan di board.

Untuk rilis produksi:

```bash
git tag v1.0 && git push origin v1.0
```

Jika ERC/DRC bersih, GitHub Release `v1.0` dibuat dengan zip lengkap, Gerber, schematic, BOM, dan file posisi.
