# Git Worktree + Claude Code: Ruang Kerja Terpisah untuk AI

> Panduan menggunakan Git Worktree sebagai cara kerja yang aman dan terisolasi ketika berkolaborasi dengan Claude Code — termasuk menangani masalah database sharing, environment variables, dan pengelolaan worktree sehari-hari.

---

## Daftar Isi

1. [Mengapa Perlu Isolasi?](#1-mengapa-perlu-isolasi)
2. [Git Worktree — Konsep Dasar](#2-git-worktree--konsep-dasar)
3. [Setup Worktree untuk Claude Code](#3-setup-worktree-untuk-claude-code)
4. [Masalah Umum dan Cara Mengatasinya](#4-masalah-umum-dan-cara-mengatasinya)
   - [.env Tidak Otomatis Tercopy](#41-env-tidak-otomatis-tercopy)
   - [Database Sharing Antar Worktree](#42-database-sharing-antar-worktree)
5. [Workflow Harian](#5-workflow-harian)
6. [Helper Scripts](#6-helper-scripts)
7. [Tips Lanjutan](#7-tips-lanjutan)
8. [Worktree: Mesin di Balik Parallel Agents](#8-worktree-mesin-di-balik-parallel-agents)

---

## 1. Mengapa Perlu Isolasi?

Saat Claude Code mengerjakan sebuah task, ia bergerak cepat — membaca, menulis, dan memodifikasi file secara aktif. Tanpa isolasi yang jelas, beberapa masalah bisa muncul:

**Konflik dengan pekerjaan aktif kamu.** Jika Claude mengerjakan task di branch yang sama dengan yang sedang kamu kerjakan, perubahan akan bercampur. Sulit membedakan mana kode dari kamu dan mana dari Claude.

**Claude terganggu oleh perubahan yang belum selesai.** File yang setengah jadi atau dependency yang belum diinstall bisa membuat Claude mengambil keputusan yang salah tentang kondisi kode saat ini.

**Tidak ada checkpoint yang jelas.** Tanpa branch terpisah, rollback jika Claude membuat kesalahan menjadi rumit.

**Git Worktree** menyelesaikan ini semua: satu repository, beberapa working directory aktif sekaligus, masing-masing di branch yang berbeda. Claude bekerja di direktorinya sendiri dan kamu tetap bekerja di direktorimu — tanpa gangguan satu sama lain.

---

## 2. Git Worktree — Konsep Dasar

### Apa Bedanya dengan Clone Biasa?

```
Clone biasa:
~/projects/calculator-app/        ← repo A (full .git/)
~/projects/calculator-app-copy/   ← repo B (full .git/)
→ Dua repo terpisah, tidak ada sinkronisasi otomatis

Git Worktree:
~/projects/calculator-app/            ← main worktree (punya .git/)
~/projects/calculator-app-issue-12/   ← linked worktree (hanya pointer ke .git/)
→ Satu .git/, dua working directory — branch tetap terhubung
```

Keuntungan worktree dibanding clone:

- Semua branch, tag, dan history tersedia di semua worktree
- Commit di worktree langsung terlihat dari worktree lain
- Tidak perlu `git remote` atau `git pull` antar worktree
- Jauh lebih hemat disk (tidak duplikasi `.git/`)

### Perintah Dasar

```bash
# Tambah worktree baru dengan branch baru
git worktree add <path> -b <branch-name>

# Tambah worktree dari branch yang sudah ada
git worktree add <path> <existing-branch>

# List semua worktree aktif
git worktree list

# Hapus worktree (setelah selesai)
git worktree remove <path>

# Hapus paksa (ada file yang belum dicommit)
git worktree remove <path> --force

# Prune worktree yang direktorinya sudah dihapus manual
git worktree prune
```

---

## 3. Setup Worktree untuk Claude Code

### Struktur Direktori yang Direkomendasikan

Tempatkan semua worktree di luar direktori utama repo, dalam satu folder parent yang sama:

```
~/projects/
├── calculator-app/              ← main repo (branch: main)
├── calculator-app-issue-12/     ← worktree untuk issue #12
├── calculator-app-issue-15/     ← worktree untuk issue #15
└── calculator-app-feat-memory/  ← worktree untuk feature baru
```

Hindari menaruh worktree di dalam direktori utama repo — bisa membingungkan tools yang scan direktori (linter, test runner, dll).

### Membuat Worktree untuk Task Claude

```bash
# Pastikan di main repo
cd ~/projects/calculator-app

# Buat worktree + branch baru sekaligus
git worktree add ../calculator-app-issue-12 -b fix/issue-12-division-by-zero

# Masuk ke worktree
cd ../calculator-app-issue-12

# Jalankan Claude Code di sini
claude
```

Di dalam Claude Code:

```
> "Baca src/Calculator.ts. Fix bug di method divide() yang
   menghasilkan Infinity ketika dibagi bilangan < 1e-15.
   Throw CalculatorError dengan pesan yang deskriptif.
   Sertakan 3 unit test."
```

### Verifikasi Worktree Aktif

```bash
git worktree list
# /home/bruce/projects/calculator-app           abc1234 [main]
# /home/bruce/projects/calculator-app-issue-12  def5678 [fix/issue-12-division-by-zero]
```

---

## 4. Masalah Umum dan Cara Mengatasinya

### 4.1 `.env` Tidak Otomatis Tercopy

Ini adalah masalah paling sering terjadi. `.env` (dan variannya seperti `.env.local`, `.env.development`) biasanya ada di `.gitignore` — artinya **tidak ikut saat worktree dibuat**. Claude Code yang berjalan di worktree baru akan langsung error karena environment variables tidak tersedia.

**Diagnosa masalah:**

```bash
cd ../calculator-app-issue-12
ls -la .env*
# → Tidak ada output. File tidak ada.

# Cek apakah memang di .gitignore
cat .gitignore | grep .env
# .env
# .env.local
# .env*.local
```

**Solusi 1: Copy manual (paling sederhana)**

```bash
# Copy dari main repo ke worktree baru
cp ~/projects/calculator-app/.env ~/projects/calculator-app-issue-12/.env

# Jika ada beberapa file .env
cp ~/projects/calculator-app/.env* ~/projects/calculator-app-issue-12/
```

**Solusi 2: Symlink (selalu sinkron dengan main)**

Gunakan symlink agar perubahan `.env` di main otomatis berlaku di semua worktree:

```bash
# Buat symlink ke .env main repo
ln -s ~/projects/calculator-app/.env ~/projects/calculator-app-issue-12/.env

# Verifikasi
ls -la ~/projects/calculator-app-issue-12/.env
# lrwxrwxrwx ... .env -> /home/bruce/projects/calculator-app/.env
```

> **Perhatian dengan symlink:** Jika Claude Code memodifikasi `.env` di worktree-nya, perubahan itu akan langsung mempengaruhi main repo juga karena mereka menunjuk ke file yang sama. Untuk menghindari ini, gunakan copy biasa jika task Claude berpotensi mengubah konfigurasi.

**Solusi 3: Script otomatis saat buat worktree**

Buat fungsi shell yang langsung mengurus `.env` setelah worktree dibuat:

```bash
# Di ~/.bashrc atau ~/.zshrc
function claude-task() {
  local issue=$1
  local repo_name=$(basename $(pwd))
  local branch="task/issue-${issue}"
  local worktree_dir="../${repo_name}-issue-${issue}"

  # Buat worktree
  git worktree add "$worktree_dir" -b "$branch"

  # Copy semua file .env yang ada
  for env_file in .env .env.local .env.development .env.test; do
    if [ -f "$env_file" ]; then
      cp "$env_file" "${worktree_dir}/${env_file}"
      echo "✓ Copied ${env_file}"
    fi
  done

  # Install dependencies jika ada package.json
  if [ -f "package.json" ]; then
    echo "→ Installing dependencies..."
    (cd "$worktree_dir" && npm install --silent)
    echo "✓ Dependencies installed"
  fi

  # Install dependencies jika ada composer.json
  if [ -f "composer.json" ]; then
    echo "→ Installing PHP dependencies..."
    (cd "$worktree_dir" && composer install --quiet)
    echo "✓ Composer dependencies installed"
  fi

  echo ""
  echo "✓ Worktree siap di: $worktree_dir"
  echo "✓ Branch: $branch"
  echo ""
  echo "Jalankan:"
  echo "  cd $worktree_dir && claude"
}
```

Penggunaan:

```bash
cd ~/projects/calculator-app
claude-task 12
# ✓ Copied .env
# ✓ Copied .env.local
# → Installing dependencies...
# ✓ Dependencies installed
# ✓ Worktree siap di: ../calculator-app-issue-12
# ✓ Branch: task/issue-12
```

---

### 4.2 Database Sharing Antar Worktree

Ini masalah yang lebih halus dan berpotensi lebih berbahaya. Secara default, **semua worktree berbagi database yang sama** karena mereka semua membaca `DATABASE_URL` yang sama dari `.env`.

#### Skenario Berbahaya

Bayangkan Bruce sedang mengerjakan fitur di main repo, dan Claude Code mengerjakan issue di worktree terpisah:

```
main repo (Bruce):
→ Sedang develop fitur baru, ada data test di DB

calculator-app-issue-12 (Claude):
→ Claude menjalankan migration untuk fix issue #12
→ Migration mengubah schema tabel calculation_history
→ BOOM: skema berubah, kode Bruce di main repo jadi broken
```

Atau lebih parah:

```
calculator-app-issue-15 (Claude):
→ Claude menjalankan seeder untuk setup test data
→ Data production/staging kamu tertimpa
```

#### Strategi 1: Database Terpisah per Worktree (Direkomendasikan)

Buat database dedicated untuk setiap worktree Claude. Ini cara paling aman.

```bash
# Buat database baru untuk worktree issue-12
createdb calculator_app_issue_12

# Atau untuk PostgreSQL via psql
psql -c "CREATE DATABASE calculator_app_issue_12;"
```

Saat membuat worktree, override `DATABASE_URL` di file `.env`-nya:

```bash
function claude-task() {
  local issue=$1
  local repo_name=$(basename $(pwd))
  local branch="task/issue-${issue}"
  local worktree_dir="../${repo_name}-issue-${issue}"
  local db_name="${repo_name//-/_}_issue_${issue}"  # calculator_app_issue_12

  # Buat worktree
  git worktree add "$worktree_dir" -b "$branch"

  # Copy .env
  cp .env "${worktree_dir}/.env"

  # Override DATABASE_URL dengan database baru
  # Untuk PostgreSQL
  sed -i "s|DATABASE_URL=.*|DATABASE_URL=postgresql://bruce:pass@localhost:5432/${db_name}|" \
    "${worktree_dir}/.env"

  # Buat database-nya
  createdb "$db_name" 2>/dev/null && echo "✓ Database '$db_name' dibuat" \
    || echo "⚠ Database '$db_name' sudah ada, dipakai ulang"

  # Jalankan migration di database baru
  echo "→ Running migrations on ${db_name}..."
  (cd "$worktree_dir" && npm run migrate 2>/dev/null \
    || npx prisma migrate deploy 2>/dev/null \
    || echo "⚠ Tidak ada migration runner yang dikenali, jalankan manual")

  echo "✓ Worktree siap di: $worktree_dir"
}
```

#### Strategi 2: Database Sama, Tapi Beri Tahu Claude

Jika membuat database baru terlalu berat, minimal dokumentasikan ini di `CLAUDE.md` agar Claude tidak sembarangan menjalankan migration atau seeder:

```markdown
# CLAUDE.md

## ⚠ Peringatan: Database Sharing

Project ini menggunakan database yang dishare dengan worktree lain.

### Yang BOLEH Claude lakukan:
- Query SELECT untuk membaca data
- INSERT/UPDATE data test yang sudah ada
- Membuat migration file (tapi jangan dijalankan)

### Yang TIDAK BOLEH Claude lakukan tanpa konfirmasi eksplisit:
- Menjalankan `npm run migrate` atau `npx prisma migrate`
- Menjalankan seeder yang menghapus atau menimpa data
- DROP TABLE atau ALTER TABLE secara langsung
- Menghapus data yang ada

Jika perlu menjalankan migration, tanyakan dulu kepada Bruce.
```

#### Strategi 3: Docker Compose per Worktree

Untuk isolasi penuh, jalankan database instance terpisah via Docker:

```yaml
# docker-compose.worktree.yml
# Tempatkan file ini di worktree
version: '3.8'
services:
  db:
    image: postgres:16
    environment:
      POSTGRES_DB: calculator_worktree
      POSTGRES_USER: bruce
      POSTGRES_PASSWORD: pass
    ports:
      - "0:5432"  # Port acak, hindari konflik
    volumes:
      - worktree_db:/var/lib/postgresql/data

volumes:
  worktree_db:
```

```bash
# Di dalam worktree
docker compose -f docker-compose.worktree.yml up -d

# Cari port yang diassign
DB_PORT=$(docker compose -f docker-compose.worktree.yml port db 5432 | cut -d: -f2)
echo "Database running on port: $DB_PORT"

# Update .env dengan port yang benar
sed -i "s|DATABASE_URL=.*|DATABASE_URL=postgresql://bruce:pass@localhost:${DB_PORT}/calculator_worktree|" .env
```

#### Strategi 4: Prefix Tabel (untuk kasus sangat spesifik)

Untuk beberapa framework yang mendukungnya, gunakan table prefix berbeda per worktree:

```bash
# .env di worktree
DB_TABLE_PREFIX=issue12_
# Dengan prefix ini: issue12_users, issue12_calculations, dst
```

Ini hanya practical jika framework kamu mendukung table prefix dan migration kamu tidak banyak.

---

### 4.3 Masalah Lain yang Perlu Diantisipasi

**Node modules / vendor tidak ada:**

```bash
# Worktree baru tidak punya node_modules
cd ../calculator-app-issue-12
npm install        # untuk Node.js
composer install   # untuk PHP
```

**Port konflik jika menjalankan dev server:**

Jika main repo dan worktree sama-sama menjalankan dev server, mereka akan berebut port yang sama.

```bash
# Di .env worktree, override port
PORT=3001          # main pakai 3000, worktree pakai 3001
```

Atau beri tahu Claude di prompt:

```
> "Jika perlu menjalankan server untuk test, pakai port 3001
   bukan 3000 karena port 3000 sudah dipakai di main repo."
```

**Cache yang saling mengganggu:**

Beberapa tools menyimpan cache di lokasi yang di-share (misalnya Jest cache):

```bash
# Jalankan test tanpa cache di worktree
npm test -- --no-cache
```

---

## 5. Workflow Harian

### Skenario: Bruce Dapat Issue Baru

Issue #12: "Calculator.divide() menghasilkan Infinity untuk input < 1e-15"

```bash
# 1. Dari main repo, buat worktree + siapkan environment
cd ~/projects/calculator-app
claude-task 12
# → Membuat worktree, copy .env, buat DB, install deps

# 2. Masuk ke worktree dan buka Claude Code
cd ../calculator-app-issue-12
claude

# 3. Di dalam Claude Code
# > "Baca CLAUDE.md dan src/Calculator.ts.
#    Fix bug divide() sesuai Issue: hasilkan Infinity untuk input < 1e-15.
#    Tambahkan test. Jangan jalankan migration apapun."
```

### Skenario: Mengerjakan Dua Issue Sekaligus

Bruce mengerjakan fitur baru di main, Claude mengerjakan dua bug fix berbeda:

```bash
# Terminal 1 — Bruce di main repo
cd ~/projects/calculator-app
# Bruce ngoding di sini

# Terminal 2 — Claude mengerjakan issue #12
cd ~/projects/calculator-app-issue-12
claude
# > "Fix bug division by zero..."

# Terminal 3 — Claude mengerjakan issue #15
cd ~/projects/calculator-app-issue-15
claude
# > "Fix bug overflow di multiply()..."
```

Ketiga terminal berjalan bersamaan tanpa konflik.

### Skenario: Review dan Merge

```bash
# Setelah Claude selesai, kembali ke main repo untuk review
cd ~/projects/calculator-app

# Lihat semua perubahan yang dibuat Claude
git diff main..fix/issue-12-division-by-zero

# Atau lebih visual dengan difftool
git difftool main..fix/issue-12-division-by-zero

# Jalankan test dari worktree-nya
cd ../calculator-app-issue-12
npm test

# Kalau OK, push dan buat PR dari main repo
cd ~/projects/calculator-app
git push origin fix/issue-12-division-by-zero
gh pr create --base main --head fix/issue-12-division-by-zero

# Setelah PR merged, bersihkan
git worktree remove ../calculator-app-issue-12
git branch -d fix/issue-12-division-by-zero

# Hapus juga database-nya
dropdb calculator_app_issue_12
```

---

## 6. Helper Scripts

Simpan semua ini di `~/.bashrc` atau `~/.zshrc`:

```bash
# ─────────────────────────────────────────────
# Git Worktree + Claude Code Helpers
# ─────────────────────────────────────────────

# Buat worktree untuk task Claude (dengan setup lengkap)
function claude-task() {
  if [ -z "$1" ]; then
    echo "Usage: claude-task <issue-number>"
    return 1
  fi

  local issue=$1
  local repo_name=$(basename $(pwd))
  local branch="task/issue-${issue}"
  local worktree_dir="../${repo_name}-issue-${issue}"
  local db_name="${repo_name//-/_}_issue_${issue}"

  # Pastikan di dalam git repo
  if ! git rev-parse --is-inside-work-tree > /dev/null 2>&1; then
    echo "❌ Bukan di dalam git repository"
    return 1
  fi

  # Cek worktree sudah ada
  if [ -d "$worktree_dir" ]; then
    echo "⚠ Worktree sudah ada di $worktree_dir"
    echo "Jalankan: cd $worktree_dir && claude"
    return 0
  fi

  echo "→ Membuat worktree: $worktree_dir (branch: $branch)"
  git worktree add "$worktree_dir" -b "$branch" || return 1

  # Copy environment files
  local env_copied=0
  for env_file in .env .env.local .env.development .env.test .env.example; do
    if [ -f "$env_file" ]; then
      cp "$env_file" "${worktree_dir}/${env_file}"
      echo "✓ Copied ${env_file}"
      env_copied=$((env_copied + 1))
    fi
  done
  [ $env_copied -eq 0 ] && echo "⚠ Tidak ada file .env ditemukan"

  # Override DATABASE_URL jika ada
  if [ -f "${worktree_dir}/.env" ] && grep -q "DATABASE_URL" "${worktree_dir}/.env" 2>/dev/null; then
    # Deteksi jenis database dari URL
    local db_url=$(grep "DATABASE_URL" "${worktree_dir}/.env" | cut -d= -f2-)
    if echo "$db_url" | grep -q "postgresql\|postgres"; then
      local new_url=$(echo "$db_url" | sed "s|/[^/]*$|/${db_name}|")
      sed -i "s|DATABASE_URL=.*|DATABASE_URL=${new_url}|" "${worktree_dir}/.env"
      createdb "$db_name" 2>/dev/null \
        && echo "✓ Database PostgreSQL '$db_name' dibuat" \
        || echo "⚠ Database '$db_name' sudah ada"
    elif echo "$db_url" | grep -q "mysql"; then
      echo "⚠ MySQL terdeteksi — buat database manual: CREATE DATABASE ${db_name};"
      echo "   Lalu update DATABASE_URL di ${worktree_dir}/.env"
    fi
  fi

  # Install dependencies
  if [ -f "package.json" ]; then
    echo "→ npm install..."
    (cd "$worktree_dir" && npm install --silent) && echo "✓ npm dependencies installed"
  fi
  if [ -f "composer.json" ]; then
    echo "→ composer install..."
    (cd "$worktree_dir" && composer install --quiet --no-interaction) \
      && echo "✓ composer dependencies installed"
  fi

  echo ""
  echo "════════════════════════════════"
  echo "✓ Worktree siap!"
  echo "  Path   : $worktree_dir"
  echo "  Branch : $branch"
  echo ""
  echo "  Jalankan:"
  echo "  cd $worktree_dir && claude"
  echo "════════════════════════════════"
}

# Hapus worktree beserta database-nya
function claude-task-done() {
  if [ -z "$1" ]; then
    echo "Usage: claude-task-done <issue-number>"
    return 1
  fi

  local issue=$1
  local repo_name=$(basename $(pwd))
  local branch="task/issue-${issue}"
  local worktree_dir="../${repo_name}-issue-${issue}"
  local db_name="${repo_name//-/_}_issue_${issue}"

  # Hapus worktree
  if [ -d "$worktree_dir" ]; then
    git worktree remove "$worktree_dir" --force
    echo "✓ Worktree dihapus: $worktree_dir"
  fi

  # Hapus branch lokal (aman jika sudah merged)
  git branch -d "$branch" 2>/dev/null \
    && echo "✓ Branch lokal dihapus: $branch" \
    || echo "⚠ Branch tidak dihapus (mungkin belum merged): $branch"

  # Hapus database
  if command -v dropdb > /dev/null 2>&1; then
    dropdb "$db_name" 2>/dev/null \
      && echo "✓ Database dihapus: $db_name" \
      || echo "⚠ Database tidak ditemukan: $db_name"
  fi

  echo "✓ Selesai cleanup issue #${issue}"
}

# List semua worktree aktif dengan status
function claude-tasks() {
  echo "Worktree aktif:"
  git worktree list
}

# Shorthand alias
alias gwl='git worktree list'
alias gwp='git worktree prune'
```

---

## 7. Tips Lanjutan

### Gunakan CLAUDE.md untuk Mencegah Kerusakan

Tempatkan instruksi penting di `CLAUDE.md` agar Claude tidak melakukan hal-hal berbahaya tanpa konfirmasi:

```markdown
# CLAUDE.md

## Konteks Worktree

Kamu sedang bekerja di isolated worktree untuk task tertentu.
Branch aktif bukan `main` — perubahan di sini tidak mempengaruhi main repo.

## Database

Worktree ini punya database sendiri (lihat DATABASE_URL di .env).
Boleh jalankan migration di database ini.
JANGAN ubah DATABASE_URL untuk menunjuk ke database lain.

## Yang Perlu Dikonfirmasi Sebelum Dilakukan

- Menghapus file yang tidak berkaitan dengan task
- Mengubah konfigurasi deployment
- Menginstall dependency baru yang belum ada di package.json
- Menjalankan script yang mempengaruhi data (seeder, fixture reset)

## Scope Task

Fokus hanya pada perubahan yang diperlukan untuk issue ini.
Jika menemukan bug lain, catat saja — jangan fix sekarang.
```

### Commit Sering dari Claude

Minta Claude commit secara bertahap, bukan sekaligus di akhir:

```
> "Kerjakan dalam commit-commit kecil:
   1. Commit pertama: tambah test yang failing
   2. Commit kedua: implementasi fix
   3. Commit ketiga: update dokumentasi jika ada"
```

Ini membuat history lebih mudah di-review dan mudah di-rollback jika ada yang salah.

### Cek Perbedaan Sebelum PR

Sebelum push, selalu review dari main repo:

```bash
# Lihat semua file yang berubah
git diff --name-only main..fix/issue-12-division-by-zero

# Lihat diff per file
git diff main..fix/issue-12-division-by-zero -- src/Calculator.ts

# Lihat commit history di worktree
git log main..fix/issue-12-division-by-zero --oneline
```

### Batasi Scope Claude via Flag

```bash
# Jalankan Claude hanya dengan akses read untuk review dulu
claude --allowedTools "Read,Glob,Grep"

# Setelah review, baru beri akses penuh untuk implementasi
claude
```

### Worktree untuk Eksperimen Bebas

Kadang kamu ingin Claude bereksperimen tanpa takut merusak apapun — misalnya mencoba refactor besar atau proof of concept:

```bash
# Buat worktree eksperimen dari branch stale
git worktree add ../calculator-app-experiment -b experiment/refactor-engine

cd ../calculator-app-experiment
claude

# > "Coba refactor Calculator class menggunakan Strategy pattern.
#    Ini eksperimen, kamu bebas mengubah apapun.
#    Kita lihat hasilnya sebelum memutuskan merge atau tidak."
```

Jika hasilnya bagus, cherry-pick commit yang relevan. Jika tidak, hapus saja worktree-nya tanpa efek samping.

---

## 8. Worktree: Mesin di Balik Parallel Agents

Semua yang sudah kamu pelajari di panduan ini tentang worktree bukan sekadar "best practice" — **ini adalah mekanisme yang digunakan baik Claude Code maupun Cursor AI untuk menjalankan parallel agents**.

Ketika kamu menjalankan beberapa agent secara bersamaan, mereka secara literal berjalan di git worktree terpisah di balik layar. Artinya setiap masalah yang sudah kita bahas di panduan ini (`.env` tidak tercopy, konflik database, tabrakan port) berlaku untuk setiap sesi parallel agent.

### Bagaimana Claude Code Menggunakan Worktree

Claude Code punya **dukungan worktree first-class** yang sudah built-in di tool:

**Flag `-w`** — buat sesi terisolasi di worktree:

```bash
# Claude membuat worktree + branch secara otomatis
claude -w fix-division-bug

# Di balik layar, Claude menjalankan:
# git worktree add .claude/worktrees/fix-division-bug -b worktree-fix-division-bug
# → Sesi dimulai di dalam direktori worktree
# → Agent bekerja dalam isolasi penuh
# → Setelah selesai, merge seperti PR biasa
```

**Sub-agent dengan `isolation: worktree`** — di definisi custom agent (`.claude/agents/`):

```markdown
---
name: feature-builder
isolation: worktree
---

You are a feature builder. Work on the assigned task in your isolated worktree.
```

Setiap kali agent ini dijalankan, Claude otomatis membuat worktree baru. Beberapa agent yang berjalan paralel masing-masing mendapat worktree sendiri.

**`/worktree` command** — manajemen worktree interaktif dari dalam Claude Code.

**`/batch` command** — menjalankan beberapa agent terisolasi worktree secara paralel dengan satu prompt. Ini cara built-in untuk bilang "jalankan 3 task ini paralel."

**Agent Teams** — ketika kamu menjalankan Claude Code Agent Teams, setiap sesi teammate mendapat worktree sendiri untuk isolasi file.

**`WorktreeCreate` / `WorktreeRemove` hooks** — Claude Code punya hook event khusus untuk lifecycle worktree. Ini tepat untuk menyelesaikan masalah `.env` copy dan setup database secara otomatis:

```json
{
  "hooks": {
    "WorktreeCreate": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "./scripts/worktree-setup.sh"
          }
        ]
      }
    ]
  }
}
```

```bash
#!/bin/bash
# scripts/worktree-setup.sh
# Dijalankan setiap kali Claude membuat worktree baru

WORKTREE_DIR=$1

# Copy file .env
for env_file in .env .env.local .env.test; do
  [ -f "$env_file" ] && cp "$env_file" "$WORKTREE_DIR/$env_file"
done

# Buat database terisolasi
DB_NAME="app_$(basename $WORKTREE_DIR | sed 's/[^a-z0-9]/_/g')"
createdb "$DB_NAME" 2>/dev/null
sed -i "s|DB_DATABASE=.*|DB_DATABASE=$DB_NAME|" "$WORKTREE_DIR/.env"

# Install dependencies
(cd "$WORKTREE_DIR" && npm install --silent 2>/dev/null)

echo "$WORKTREE_DIR"
```

**Desktop app** — Claude Code Desktop otomatis mengisolasi setiap sesi baru di worktree sendiri.

### Bagaimana Cursor AI Menggunakan Worktree

**Parallel Agents** Cursor (Agents Window di Cursor 3) menggunakan mekanisme yang persis sama:

```
Kamu assign task ke beberapa agent di Agents Window:
  Agent A: "Fix bug #12"
  Agent B: "Tambah fitur reporting"
  Agent C: "Refactor module auth"

Di balik layar, Cursor menjalankan:
  git worktree add ../agent-a -b feature/agent-a
  git worktree add ../agent-b -b feature/agent-b
  git worktree add ../agent-c -b feature/agent-c
```

Setiap agent bekerja di worktree sendiri di branch sendiri. Mereka tidak bisa menyentuh file agent lain. Setelah agent selesai, kamu review dan merge branch mereka seperti PR biasa.

### Mengapa Ini Penting

Memahami bahwa worktree menggerakkan parallel agents mengubah cara kamu memandang kedua tool:

**1. Masalah dari Section 4 berlaku untuk setiap sesi agent.**

Ketika Claude Code atau Cursor men-spawn parallel agent di worktree, worktree itu punya masalah yang sama seperti yang sudah kita bahas:
- File `.env` tidak tercopy (gunakan `WorktreeCreate` hooks untuk memperbaikinya)
- Database di-share secara default (konfigurasi database per-worktree)
- Dependencies tidak terinstall (gunakan hooks atau setup scripts)
- Port dev server konflik (assign port berbeda per worktree)

**2. Kamu bisa mengontrol behavior agent melalui worktree hooks.**

Hook `WorktreeCreate` Claude Code memungkinkan kamu menjalankan setup script setiap kali agent dimulai. Artinya kamu bisa:
- Auto-copy file `.env`
- Buat database terisolasi
- Install dependencies
- Setup environment sebelum agent mulai coding

**3. Cleanup setelah agent sama dengan cleanup worktree.**

```bash
# Lihat semua worktree agent yang aktif
git worktree list

# Bersihkan worktree agent yang sudah selesai
git worktree remove .claude/worktrees/fix-division-bug
git branch -d worktree-fix-division-bug
```

**4. Flag `-w` membuat manual worktree setup jadi opsional.**

Jika kamu menggunakan Claude Code, kamu tidak selalu butuh helper scripts dari Section 6. Flag `-w` dan `WorktreeCreate` hooks menangani sebagian besar setup secara otomatis. Tapi memahami apa yang terjadi di balik layar (panduan ini) membantu kamu debug ketika ada masalah.

### Ringkasan: Worktree di Mana-mana

```
Kamu buat worktree manual       Claude Code flag -w       Cursor Parallel Agents
          │                              │                            │
          ▼                              ▼                            ▼
  git worktree add <path>      git worktree add             git worktree add
  -b <branch>                  .claude/worktrees/<name>     ../agent-<name>
                               -b worktree-<name>          -b feature/agent-<name>
          │                              │                            │
          ▼                              ▼                            ▼
  Masalah yang sama: .env tidak tercopy, DB di-share, deps hilang, port konflik
          │                              │                            │
          ▼                              ▼                            ▼
  Solusi yang sama: setup scripts, hooks, database terisolasi
```

**Mekanisme yang sama, entry point berbeda.** Baik kamu buat worktree manual, biarkan Claude Code melakukannya dengan `-w`, atau biarkan Cursor menangani via Parallel Agents — semuanya git worktree di balik layar. Itulah kenapa module ini ada di urutan pertama: semua yang lain dibangun di atas fondasi ini.
