# PROJECT_NAME

[English](README.md) | **Bahasa Indonesia**

Project hardware KiCad 10.

**Daftar isi**

<!-- template:start -->
- [Menggunakan template ini](#menggunakan-template-ini)
<!-- template:end -->
- [Struktur](#struktur)
- [Menambah part dan library](#menambah-part-dan-library)
  - [A. Library bawaan KiCad](#a-library-bawaan-kicad)
  - [B. Repository git library KiCad (submodule)](#b-repository-git-library-kicad-submodule)
  - [C. Download dari vendor (zip)](#c-download-dari-vendor-zip)
  - [D. Gambar sendiri](#d-gambar-sendiri)
  - [Setelah menambah part](#setelah-menambah-part)
- [Asisten AI (MCP)](#asisten-ai-mcp)
- [CI & rilis](#ci--rilis)

<!-- template:start -->
## Menggunakan template ini

1. Klik **Use this template → Create a new repository** di GitHub. Nama repository
   akan menjadi nama project KiCad (mis. `sensor-board`).
2. Workflow **KiCad** berjalan sekali dengan judul *"Updating the project name to sensor-board, then checking the build"*:
   - job **Rename project**: mengganti `PROJECT_NAME` dengan nama repository di nama file maupun isi file
     (`sources/sensor-board/sensor-board.kicad_pro`, `libraries/sensor-board.kicad_sym`, dst.),
     membuat ulang UUID root schematic, menghapus bagian ini dari kedua README, lalu di-commit
     oleh `github-actions[bot]`
   - job **Build**: ERC, DRC, dan output fabrikasi untuk project yang sudah di-rename
3. `git pull`, lalu buka file `.kicad_pro` di KiCad.

Setelah itu, setiap push hanya menampilkan satu run *"Build: …"* (job **Rename project** di-skip).

Clone lokal tanpa GitHub? Jalankan sendiri:

```bash
scripts/init.sh nama-project   # tanpa argumen: pakai nama folder repository
```

> Jika commit rename ditolak, buka **Settings → Actions → General → Workflow permissions**,
> pilih **Read and write permissions**, lalu jalankan workflow secara manual
> (tab Actions → KiCad → Run workflow).
<!-- template:end -->

## Struktur

```
sources/PROJECT_NAME/        KiCad project (.kicad_pro/.kicad_sch/.kicad_pcb) + lib tables
libraries/
  PROJECT_NAME.kicad_sym     simbol khusus project
  PROJECT_NAME.pretty/       footprint khusus project
  PROJECT_NAME.3dshapes/     model 3D (STEP/WRL)
  external/<nama>/           library eksternal (git submodule)
scripts/                     init.sh, add/remove-library.sh, mcp-kicad.sh
.mcp.json, .vscode/, .cursor/ konfigurasi MCP untuk asisten AI
AGENTS.md, CLAUDE.md         instruksi untuk AI agent
.github/workflows/kicad.yml  ERC, DRC, dan output fabrikasi
```

Library project sudah terdaftar di `sym-lib-table` / `fp-lib-table` project dengan path
`${KIPRJMOD}/../../libraries/...`, jadi tetap jalan setelah di-clone di mana pun.
Untuk model 3D, isi path footprint dengan `${KIPRJMOD}/../../libraries/PROJECT_NAME.3dshapes/<file>.step`.

Title block memakai text variable `${PROJECT}`, `${REVISION}` dan `${CURRENT_DATE}`.
`REVISION` bernilai `dev` di KiCad dan diisi otomatis oleh CI dari tag git / commit.

## Menambah part dan library

Tanyakan dulu: **part-nya ada di mana?** Jawabannya menentukan jalurnya.

| Part ada… | Jalur | Masuk ke |
| --- | --- | --- |
| di library bawaan KiCad | **A.** langsung pakai | – |
| di repository git library KiCad | **B.** `scripts/add-library.sh <url>` | `libraries/external/<nama>/` (submodule) |
| di file download vendor (SnapEDA, Ultra Librarian, zip dari Mouser/DigiKey) | **C.** import | `libraries/PROJECT_NAME.*` |
| tidak ada di mana pun | **D.** gambar sendiri | `libraries/PROJECT_NAME.*` |

### A. Library bawaan KiCad

Selalu cek di sini dulu: resistor, kapasitor, LED, pin header, regulator umum, header 40-pin
Raspberry Pi, ... Tekan **A** di schematic editor lalu cari. Library bawaan ikut terpasang bersama
KiCad (dan di container CI), jadi tidak perlu didaftarkan.

### B. Repository git library KiCad (submodule)

```bash
scripts/add-library.sh https://github.com/<owner>/<kicad-lib>.git          # -> libraries/external/<kicad-lib>
scripts/add-library.sh https://github.com/<owner>/<kicad-lib>.git mylib -b main
git commit -m "Add mylib library"
```

Script ini menjalankan `git submodule add`, lalu mendaftarkan setiap `*.kicad_sym` dan `*.pretty`
di dalam submodule ke `sym-lib-table` / `fp-lib-table` project (nickname = nama file, path lewat
`${KIPRJMOD}`). Nickname yang sudah ada dilewati. Buka ulang project di KiCad setelahnya.

Submodule membuat repository tetap kecil, mengunci versi library per revisi board, dan bisa di-update
kalau vendor memperbaiki sesuatu. **Jangan pernah mengedit file di `libraries/external/`**; kalau
perlu mengubah satu part, copy ke library project (jalur C/D).

```bash
git clone --recursive <repo-url>              # clone beserta library
git submodule update --init --recursive       # setelah clone / pull biasa
git submodule update --remote libraries/external/mylib # update library ke commit terbaru
```

Untuk menghapus library:

```bash
scripts/remove-library.sh mylib
git commit -m "Remove mylib library"
```

Script ini melepas dan menghapus submodule (termasuk salinannya di `.git/modules`) serta menghapus
entri-nya dari lib table. Script menolak jalan selama schematic atau board masih memakai simbol atau
footprint dari library tersebut; ganti komponennya dulu, atau pakai `--force`.

Gunakan URL `https://` agar CI bisa mengambilnya. Untuk repository library private, tambahkan
repository secret `SUBMODULE_TOKEN` (PAT dengan akses read); checkout di CI otomatis memakainya.

### C. Download dari vendor (zip)

Import file-nya ke library milik project, yang sudah terdaftar:

| File | Masukkan ke | Caranya |
| --- | --- | --- |
| Simbol (`.kicad_sym`) | `libraries/PROJECT_NAME.kicad_sym` | Symbol Editor → pilih library `PROJECT_NAME` → **File → Import Symbol** |
| Footprint (`.kicad_mod`) | `libraries/PROJECT_NAME.pretty/` | copy file-nya, atau Footprint Editor → **File → Import Footprint** |
| Model 3D (`.step`) | `libraries/PROJECT_NAME.3dshapes/` | copy file-nya; di **Properties → 3D Models** footprint isi `${KIPRJMOD}/../../libraries/PROJECT_NAME.3dshapes/<file>.step` |

Lalu isi field **Footprint** di simbol dengan `PROJECT_NAME:<footprint>`.

> Selalu cek footprint hasil download dengan datasheet (ukuran pad, jarak pin, pin 1). Kesalahan
> footprint baru ketahuan setelah board datang dari pabrik.

### D. Gambar sendiri

1. Symbol Editor → **New Symbol** di library `PROJECT_NAME`. Isi **jenis elektrik** setiap pin
   (Input, Output, Power input, ...) dengan benar: ERC memakainya.
2. Footprint Editor → **New Footprint** di `PROJECT_NAME.pretty`, mengikuti *recommended land pattern*
   di datasheet (atau **Footprint Wizard** untuk package standar seperti QFN atau SOIC).
3. Isi field **Footprint** di simbol dan, kalau ada, model 3D seperti di jalur C.

### Setelah menambah part

1. Pasang di schematic lalu jalankan **Update PCB from Schematic** (F8).
2. Jalankan ERC dan DRC (atau minta asisten AI).
3. `git push`. Kalau CI gagal padahal di laptop lolos, biasanya path library masih absolut
   (`/Users/...`) dan belum relatif lewat `${KIPRJMOD}`.

## Asisten AI (MCP)

Repository ini sudah menyertakan server [Model Context Protocol](https://modelcontextprotocol.io)
untuk KiCad, [kicad-mcp-pro](https://github.com/oaslananka/kicad-mcp-pro), yang langsung mengarah ke project ini:

| Client | Konfigurasi |
| --- | --- |
| Claude Code | `.mcp.json` (setujui server `kicad` saat pertama kali dijalankan) |
| VS Code / Copilot | `.vscode/mcp.json` |
| Cursor | `.cursor/mcp.json` |
| Client lain | perintah `bash scripts/mcp-kicad.sh` (stdio) |

Kebutuhan: [uv](https://docs.astral.sh/uv/getting-started/installation/) (`uvx`) dan KiCad 10
dengan `kicad-cli` di `PATH`. Di Windows, jalankan lewat Git Bash / WSL.
Server mencari `sources/*/*.kicad_pro` sendiri, jadi tidak ada yang perlu diubah per project.

Perilaku server bisa diatur dengan environment variable yang dibaca `scripts/mcp-kicad.sh`:
`KICAD_MCP_OPERATING_MODE` (`readonly`, `write` *(default)*, `manufacturing`),
`KICAD_MCP_PROFILE` (`default`, `review`, `build`, `release`, `full`, ...) dan
`KICAD_MCP_PACKAGE` (mis. `kicad-mcp-pro==3.35.0` untuk mengunci versi).

Aturan project untuk AI agent ada di [`AGENTS.md`](AGENTS.md) (di-import oleh `CLAUDE.md`).

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
