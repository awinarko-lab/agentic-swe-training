# Setup Project & Guardrails — Sebelum Mulai Bekerja dengan AI

> Panduan menyiapkan project sebelum pertama kali menggunakan AI coding assistant. Mencakup privasi, keamanan, konvensi kode, testing, dan lapisan permission. Wajib dibaca sebelum lanjut ke module lain.

---

## Daftar Isi

1. [Mengapa Ini Penting?](#1-mengapa-ini-penting)
2. [Privasi & Keamanan](#2-privasi--keamanan)
   - [2.1 AI Bisa Baca Semua File Kamu](#21-ai-bisa-baca-semua-file-kamu)
   - [2.2 Ignore Files per Tool](#22-ignore-files-per-tool)
   - [2.3 Protect Secrets](#23-protect-secrets)
   - [2.4 Audit Apa yang Terkirim ke AI](#24-audit-apa-yang-terkirim-ke-ai)
3. [Style Guide & Konvensi](#3-style-guide--konvensi)
   - [3.1 CLAUDE.md — Konstitusi Project untuk Claude Code](#31-claudemd--konstitusi-project-untuk-claude-code)
   - [3.2 .cursorrules — Konstitusi Project untuk Cursor](#32-cursorrules--konstitusi-project-untuk-cursor)
   - [3.3 Konvensi yang Harus Didokumentasikan](#33-konvensi-yang-harus-didokumentasikan)
   - [3.4 Satu Sumber Kebenaran](#34-satu-sumber-kebenaran)
4. [Testing Guardrails](#4-testing-guardrails)
   - [4.1 Pre-commit Hooks](#41-pre-commit-hooks)
   - [4.2 Instruksi Testing di Config AI](#42-instruksi-testing-di-config-ai)
   - [4.3 CI Pipeline Tetap Wajib](#43-ci-pipeline-tetap-wajib)
   - [4.4 Plan Mode — Baca Dulu, Baru Ubah](#44-plan-mode--baca-dulu-baru-ubah)
5. [Lapisan Permission (Defense in Depth)](#5-lapisan-permission-defense-in-depth)
   - [5.1 Layer 1: OS-Level](#51-layer-1-os-level)
   - [5.2 Layer 2: Config-Level](#52-layer-2-config-level)
   - [5.3 Layer 3: Instruction-Level](#53-layer-3-instruction-level)
   - [5.4 Rangkuman 3 Layer](#54-rangkuman-3-layer)
6. [Checklist — Sebelum Mulai](#6-checklist--sebelum-mulai)
7. [Referensi](#7-referensi)

---

## 1. Mengapa Ini Penting?

Bayangkan kamu hire junior developer baru. Sebelum dia mulai coding, kamu pasti akan:

- **Brief dia** tentang konvensi project
- **Berikan akses** ke repo, tapi tidak ke production secrets
- **Jelaskan workflow** — gimana cara test, gimana cara deploy
- **Set expectations** — apa yang boleh, apa yang tidak

AI coding assistant sama seperti junior developer baru — tapi lebih berbahaya karena:

1. **Dia baca semua file di directory project**, termasuk yang seharusnya rahasia
2. **Dia bisa menjalankan perintah terminal** dengan permission kamu
3. **Dia tidak punya common sense** — akan mengikuti instruksi secara literal
4. **Dia sangat cepat** — bisa merusak banyak hal sebelum kamu sempat review

Setup yang baik memakan waktu di awal, tapi menyelamatkan kamu dari masalah besar nanti.

---

## 2. Privasi & Keamanan

### 2.1 AI Bisa Baca Semua File Kamu

Ini fakta yang sering terlupakan: **`.gitignore` tidak melindungi kamu dari AI**.

`.gitignore` mencegah file masuk ke git repository. Tapi AI coding assistant membaca **file system lokal** — bukan git index. Artinya:

```
File yang ADA di .gitignore:
├── .env                    ← AI bisa baca ini
├── .env.production         ← AI bisa baca ini
├── id_rsa                  ← AI bisa baca ini
├── deploy/credentials.json ← AI bisa baca ini
└── secrets/                ← AI bisa baca semua isi folder ini
```

Semua file itu bisa terkirim ke server AI (Anthropic, OpenAI, dll.) sebagai bagian dari context.

**Apa artinya?**
- API keys bisa bocor
- Database credentials bisa terekspos
- Konfigurasi infrastruktur internal bisa terbaca
- Data yang sifatnya compliance-sensitive bisa terkirim

### 2.2 Ignore Files per Tool

Setiap AI tool punya mekanisme ignore sendiri. Kamu perlu setup **semua** tool yang dipakai tim:

#### Claude Code — `.claudeignore`

```bash
# Buat file di root project
touch .claudeignore
```

```gitignore
# .claudeignore — file dan folder yang Claude Code TIDAK boleh baca

# Secrets
.env
.env.*
!.env.example

# Infrastructure
deploy/
terraform/
ansible/
helm/

# Private keys
*.pem
*.key
id_rsa*

# Database dumps
*.sql
*.dump

# Logs (bisa berisi sensitive data)
logs/
*.log

# Vendor internal yang tidak boleh dishare
internal-tools/
proprietary/
```

#### Cursor — `.cursorignore`

```bash
touch .cursorignore
```

Isinya sama dengan `.claudeignore`. Cursor menghormati file ini untuk mencegah agent membaca file sensitif.

#### Claude Code — `.claude/settings.json`

Bisa juga restrict via settings file:

```json
{
  "permissions": {
    "deny": [
      "Bash(git push --force*)",
      "Bash(rm -rf *)",
      "Bash(DROP TABLE*)",
      "Bash(curl*production*)"
    ]
  }
}
```

> **Penting:** Hanya sebagian kecil project yang menggunakan AI assistant yang mengkonfigurasi ignore rules. Jangan jadi salah satunya.

### 2.3 Protect Secrets

**Langkah wajib sebelum menggunakan AI di project:**

```bash
# 1. Cek apakah ada secrets di git history
git log --all --diff-filter=A -- '*.env' '*.key' '*.pem' 'credentials*'

# 2. Pastikan .env files ada di .gitignore
cat .gitignore | grep -E '\.env'

# 3. Pastikan .env files juga ada di .claudeignore dan .cursorignore
cat .claudeignore | grep -E '\.env'
cat .cursorignore | grep -E '\.env'

# 4. Jika ada secrets yang pernah di-commit, ROTATE sekarang
# API keys, database passwords, JWT secrets — anggap sudah bocor
```

**Best practice:**

- Gunakan `.env.example` sebagai template (ini BOLEH dibaca AI)
- Simpan secrets di **environment variable** atau **secret manager**, bukan di file
- Jika project menggunakan Docker, gunakan Docker secrets atau vault
- Rotate semua secrets yang pernah ada di git history

### 2.4 Audit Apa yang Terkirim ke AI

Untuk Claude Code, kamu bisa cek apa yang terkirim:

```bash
# Jalankan Claude Code dengan verbose logging
claude --debug

# Atau cek transcript
claude sessions list
```

Untuk Cursor, cek di:
- **Settings → General → Privacy** — bisa melihat dan menghapus data yang terkirim
- **Activity log** — untuk melihat apa yang dilakukan agent

---

## 3. Style Guide & Konvensi

### 3.1 CLAUDE.md — Konstitusi Project untuk Claude Code

`CLAUDE.md` adalah file yang dibaca Claude di **setiap session**. Ini adalah tempat paling penting untuk mendefinisikan bagaimana AI harus bekerja di project kamu.

```bash
# Generate template awal (opsional)
claude
> /init
```

**Contoh CLAUDE.md yang baik:**

```markdown
# Project: payment-gateway

## Build & Test
- Build: `make build`
- Test: `make test`
- Lint: `make lint`
- Run semua sebelum commit: `make ci`

## Tech Stack
- Language: Go 1.22
- Database: PostgreSQL + golang-migrate untuk migration
- HTTP Framework: chi router
- Queue: Redis + asynq

## Conventions
- Semua API response pakai envelope format: `{data, error, meta}`
- Error handling: return error ke caller, JANGAN gunakan panic()
- Naming: camelCase untuk variable, PascalCase untuk export, snake_case untuk DB column
- Commit message: conventional commits (`feat:`, `fix:`, `docs:`, dll.)

## Architecture
- Double-entry accounting: lihat `docs/accounting.md`
- QRIS payment flow: lihat `docs/qris-flow.md`
- Service layer pattern: handler → service → repository

## Testing
- Selalu tulis test untuk kode baru
- Test file di `*_test.go` sebelah file yang ditest
- Mock external service, JANGAN mock database (integration test)
- Target coverage: minimal 80% untuk service layer

## Yang TIDAK BOLEH dilakukan tanpa konfirmasi
- Menjalankan migration di database production
- Mengubah file config deployment
- Menghapus file yang tidak terkait task saat ini
- Install dependency yang belum ada di go.mod
```

**Tips CLAUDE.md:**

- **Jangan terlalu panjang** — Claude membaca ini di setiap request. 100-200 baris ideal.
- **Front-load yang penting** — build/test commands di atas, conventions di tengah, restrictions di bawah.
- **Spesifik, bukan generik** — "gunakan conventional commits" lebih baik dari "tulis commit yang bagus".
- **Update saat project berubah** — tambah tech stack baru, ubah conventions, dll.

### 3.2 .cursorrules — Konstitusi Project untuk Cursor

File `.cursorrules` di root project berfungsi sama seperti CLAUDE.md tapi untuk Cursor:

```bash
touch .cursorrules
```

```markdown
You are an expert developer working on the payment-gateway project.

## Rules
- Always run `make lint` and `make test` before suggesting changes
- Follow Go naming conventions and project patterns
- Use chi router for new HTTP endpoints
- All responses use envelope format: {data, error, meta}
- Write tests for new code (target 80% coverage on service layer)
- Use conventional commits: feat:, fix:, docs:, refactor:

## Restrictions
- Never modify deployment configs without asking
- Never run database migrations on production
- Never install packages not in go.mod
- Always ask before deleting files

## Architecture
- Double-entry accounting pattern (see docs/accounting.md)
- Handler → Service → Repository layering
- Integration tests with real database, not mocks
```

> **Catatan:** Untuk rules yang lebih kompleks di Cursor, gunakan `.cursor/rules/*.mdc` — mendukung multiple rule files per concern.

### 3.3 Konvensi yang Harus Didokumentasikan

Tidak semua konvensi perlu ditulis. Fokus pada yang **AI sering salah**:

| Wajib tulis | Opsional |
|---|---|
| Build, test, lint commands | Comment style |
| Commit message format | Variable naming (sudah ada linter) |
| Framework & versi yang dipakai | File organization |
| Pattern arsitektur (layering, dll.) | Import order |
| Error handling approach | Spacing / formatting |
| Database approach (ORM? raw query?) | Git branching strategy |
| Testing approach (unit? integration? mock apa?) | Deployment process |
| **Restrictions** — apa yang tidak boleh dilakukan | |

**Restrictions lebih penting daripada conventions.** AI lebih sering melakukan hal yang tidak boleh daripada salah format.

### 3.4 Satu Sumber Kebenaran

Jika tim menggunakan Claude Code DAN Cursor, kamu punya masalah duplikasi:

```
CLAUDE.md        ← konvensi untuk Claude Code
.cursorrules     ← konvensi untuk Cursor
AGENTS.md        ← konvensi untuk Codex/OpenCode (kalau nanti pakai)
```

**Strategi untuk menghindari drift:**

1. **Pilih satu sebagai source of truth** — misalnya CLAUDE.md
2. **Generate yang lain secara otomatis** — script yang convert CLAUDE.md → .cursorrules
3. **Atau: pisahkan concern** — bagian yang sama di semua file, tool-specific di file masing-masing

Contoh struktur:

```
docs/
└── ai-conventions.md      ← SOURCE OF TRUTH (human-maintained)

CLAUDE.md                   ← generate dari ai-conventions.md + claude-specific
.cursorrules                ← generate dari ai-conventions.md + cursor-specific
```

Script generator (opsional):

```bash
#!/bin/bash
# scripts/generate-ai-config.sh

CONVENTIONS="docs/ai-conventions.md"

# Claude Code: conventions + claude-specific rules
cat "$CONVENTIONS" > CLAUDE.md
cat docs/claude-specific.md >> CLAUDE.md

# Cursor: conventions + cursor-specific rules
cat "$CONVENTIONS" > .cursorrules
cat docs/cursor-specific.md >> .cursorrules

echo "✓ Generated CLAUDE.md and .cursorrules from $CONVENTIONS"
```

---

## 4. Testing Guardrails

### 4.1 Pre-commit Hooks

Pre-commit hooks memastikan setiap commit (dari manusia atau AI) melewati quality gate yang sama.

```bash
# Install pre-commit framework
pip install pre-commit

# Buat .pre-commit-config.yaml
```

```yaml
# .pre-commit-config.yaml
repos:
  # Linting
  - repo: https://github.com/golangci/golangci-lint
    rev: v1.59.0
    hooks:
      - id: golangci-lint

  # Formatting
  - repo: https://github.com/psf/black
    rev: 24.8.0
    hooks:
      - id: black

  # Secret detection — PENTING!
  - repo: https://github.com/Yelp/detect-secrets
    rev: v1.4.0
    hooks:
      - id: detect-secrets
        args: ['--baseline', '.secrets.baseline']

  # Conventional commits
  - repo: https://github.com/compilerla/conventional-pre-commit
    rev: v3.1.0
    hooks:
      - id: conventional-pre-commit
        stages: [commit-msg]
```

```bash
# Install hooks
pre-commit install
pre-commit install --hook-type commit-msg

# Test bahwa hooks berjalan
pre-commit run --all-files
```

**Mengapa ini penting untuk AI?**

AI tidak selalu mengikuti formatting rules. Pre-commit hooks memaksa konsistensi terlepas dari siapa yang menulis kode — manusia atau AI.

### 4.2 Instruksi Testing di Config AI

Tambahkan instruksi testing ke CLAUDE.md dan .cursorrules:

```markdown
## Testing Rules

1. WAJIB tulis test untuk setiap kode baru atau perubahan logika
2. Jalankan test setelah setiap perubahan — pastikan semua pass
3. Jika ada test yang failing, JANGAN lanjut ke task berikutnya
4. Target coverage: 80% untuk service layer
5. Mock external service (API call, email), JANGAN mock database
6. Test file: `*_test.go` (Go), `*.test.ts` (TypeScript), `*_test.py` (Python)
```

### 4.3 CI Pipeline Tetap Wajib

**AI-generated code melewati CI yang sama dengan human code.** Tidak ada pengecualian.

```yaml
# .github/workflows/ci.yml
name: CI
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: make lint
      - run: make test
      - run: make build
```

> **Jangan pernah skip CI** untuk PR dari AI — bahkan jika kamu "yakin" kodenya benar.

### 4.4 Plan Mode — Baca Dulu, Baru Ubah

Sebelum biarkan AI mengubah apapun, minta dia **berencana dulu**:

**Claude Code:**

```bash
# Jalankan di plan mode (read-only, tidak ada perubahan file)
claude --plan

# Atau restricted tools
claude --allowedTools "Read,Glob,Grep"
```

**Cursor:**

```
# Di chat, minta plan dulu:
> "Analisis bug ini dan buat plan. JANGAN edit file apapun — hanya jelaskan
   langkah-langkah yang akan kamu ambil dan file apa yang akan berubah."
```

**Mengapa ini berguna?**

- Kamu bisa review plan sebelum AI mengubah apapun
- AI sering menemukan insight saat merencanakan yang mengubah approach-nya
- Mengurangi wasted work jika AI salah arah dari awal

---

## 5. Lapisan Permission (Defense in Depth)

Keamanan AI bukan satu hal — itu beberapa lapisan. Semakin banyak lapisan, semakin aman.

### 5.1 Layer 1: OS-Level (Terkuat)

Enforcement di level sistem operasi. Bahkan jika AI "nakal", dia tidak bisa melewati ini.

| Metode | Contoh | Kapan pakai |
|--------|--------|-------------|
| **Docker container** | Jalankan AI di container terisolasi | Project high-risk |
| **User terpisah** | `ai-user` tanpa sudo access | Setup yang simple |
| **Read-only filesystem** | Mount project sebagai read-only untuk plan mode | Review phase |
| **Network restrictions** | Block outbound ke production endpoint | Saat testing |

Contoh Docker untuk isolation:

```bash
# Jalankan Claude Code di container terisolasi
docker run -it --rm \
  -v $(pwd):/workspace \
  --network none \          # no network access
  --read-only \             # read-only filesystem (except workspace)
  claude-code:latest
```

> **Note:** Untuk kebanyakan project, Layer 1 terlalu berat. Tapi penting untuk tau opsi ini ada.

### 5.2 Layer 2: Config-Level (Praktis)

Enforcement melalui konfigurasi tool. Ini lapisan yang **paling seimbang** antara keamanan dan kemudahan.

#### Claude Code — `.claude/settings.json`

```json
{
  "permissions": {
    "allow": [
      "Read",
      "Glob",
      "Grep",
      "Bash(make test*)",
      "Bash(make lint*)",
      "Bash(git diff*)",
      "Bash(git log*)",
      "Bash(git add*)",
      "Bash(npm test*)"
    ],
    "deny": [
      "Bash(rm -rf *)",
      "Bash(git push --force*)",
      "Bash(git push origin main*)",
      "Bash(DROP TABLE*)",
      "Bash(DROP DATABASE*)",
      "Bash(curl*production*)",
      "Bash(curl*staging*)"
    ]
  }
}
```

#### Claude Code — Hooks

Hooks adalah **shell scripts yang berjalan secara deterministik** pada event tertentu. Ini lebih kuat dari instruksi di CLAUDE.md.

```bash
# Buat guard script
mkdir -p .claude/hooks
```

```bash
#!/bin/bash
# .claude/hooks/guard.sh — Blokir perintah berbahaya

INPUT=$(cat)
CMD=$(echo "$INPUT" | jq -r '.tool_input.command // empty')

# Blokir perintah destruktif
if echo "$CMD" | grep -qiE "DROP TABLE|DROP DATABASE|rm -rf|git push --force"; then
  echo "🚫 BLOCKED: Perintah ini tidak diizinkan." >&2
  echo "Jika benar-benar perlu, jalankan manual di terminal." >&2
  exit 2  # exit 2 = blokir + kirim feedback ke AI
fi

exit 0
```

```json
// .claude/settings.json — daftarkan hook
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "./.claude/hooks/guard.sh"
          }
        ]
      }
    ]
  }
}
```

> **Perbedaan penting:** Aturan di CLAUDE.md dipatuhi ~70% waktu. Hooks diblokir **100%** waktu. Untuk safety-critical rules, selalu gunakan hooks.

#### Cursor — Permission Settings

Di Cursor, atur melalui:
- **Settings → Features → Agent** — toggle mana yang boleh agent lakukan
- **Approval mode** — pilih "Auto-accept reads, ask for writes" atau lebih ketat

### 5.3 Layer 3: Instruction-Level (Terlemah)

Ini adalah instruksi di CLAUDE.md, .cursorrules, atau prompt. AI memperlakukannya sebagai *context*, bukan *law*.

```markdown
# Di CLAUDE.md atau .cursorrules

## ⚠️ Safety Rules
- JANGAN pernah menjalankan migration tanpa konfirmasi
- JANGAN pernah push ke main branch
- JANGAN pernah menghapus file yang tidak terkait task
- JANGAN pernah install dependency baru tanpa konfirmasi
- Selalu tanya sebelum menjalankan perintah yang mengubah database
```

**Kenapa masih perlu ini?** Karena:
- Membantu AI membuat keputusan yang lebih baik secara default
- Mengurangi frekuensi hal berbahaya (dari sering → jarang)
- Sebagai "human-readable documentation" untuk developer lain di tim

Tapi **jangan andalkan ini sebagai satu-satunya pertahanan**.

### 5.4 Rangkuman 3 Layer

```
┌─────────────────────────────────────────────────┐
│  Layer 3: Instruction (CLAUDE.md, .cursorrules) │  ← ~70% compliance
│  "Jangan push ke main"                           │
├─────────────────────────────────────────────────┤
│  Layer 2: Config (settings.json, hooks, deny)   │  ← ~100% enforcement
│  Blokir git push --force via hook                │
├─────────────────────────────────────────────────┤
│  Layer 1: OS (Docker, user isolation, network)  │  ← 100% enforcement
│  Container tanpa network access                  │
└─────────────────────────────────────────────────┘
```

| | Layer 3 | Layer 2 | Layer 1 |
|---|---|---|---|
| **Keamanan** | Rendah | Tinggi | Sangat tinggi |
| **Kemudahan setup** | Mudah | Sedang | Sulit |
| **Recommended untuk** | Semua project | Semua project | High-risk / enterprise |

**Rekomendasi praktis:**
- **Semua project:** Layer 2 + Layer 3 (config + instruksi)
- **Production / enterprise:** Tambah Layer 1 (OS isolation)
- **Never:** Hanya Layer 3 (instruksi saja tidak cukup)

---

## 6. Checklist — Sebelum Mulai

Gunakan checklist ini untuk setiap project baru sebelum menggunakan AI:

### 🔒 Keamanan

- [ ] Buat `.claudeignore` dengan daftar file sensitif
- [ ] Buat `.cursorignore` dengan daftar file sensitif
- [ ] Pastikan `.env` dan secrets **tidak** bisa dibaca AI
- [ ] Setup `.claude/settings.json` dengan deny rules
- [ ] (Opsional) Setup hooks untuk blocking command berbahaya
- [ ] Rotate secrets yang pernah di-commit ke git

### 📏 Konvensi

- [ ] Buat `CLAUDE.md` dengan build/test commands, conventions, dan restrictions
- [ ] Buat `.cursorrules` dengan rules yang sama
- [ ] Dokumentasikan pattern arsitektur yang harus diikuti
- [ ] Dokumentasikan apa yang **tidak boleh** dilakukan AI

### 🧪 Testing

- [ ] Setup pre-commit hooks (lint + secret detection + commit format)
- [ ] Tambahkan instruksi testing di config AI
- [ ] Pastikan CI pipeline aktif dan tidak bisa di-skip
- [ ] Test plan mode sebelum biarkan AI edit file

### ✅ Verifikasi

- [ ] Jalankan AI di project dengan **plan mode** — pastikan dia memahami konvensi
- [ ] Minta AI tulis kode sederhana — cek apakah mengikuti style guide
- [ ] Minta AI tulis test — cek apakah test approach sesuai
- [ ] Coba trigger perintah berbahaya — pastikan diblokir oleh hooks/settings

---

## 7. Referensi

- [AI Ignore Rules: Protect Your Secrets](https://appga.pl/2025/11/22/ai-ignore-rules-protect-your-secrets-when-using-code-assistants/)
- [gitignore Won't Save Your Secrets from AI Coding Agents](https://medium.com/ai-mindset/gitignore-wont-save-your-secrets-from-ai-coding-agents-35dddf892061)
- [Claude Code Permissions Docs](https://code.claude.com/docs/en/agent-sdk/permissions)
- [AI Coding Agent Security: Practical Guardrails](https://dev.to/maxkrivich/ai-coding-agent-security-practical-guardrails-for-claude-code-copilot-and-codex-och)
- [Claude Code Hooks Guide](https://code.claude.com/docs/en/hooks-guide)
- [CLAUDE.md Best Practices](https://uxplanet.org/claude-md-best-practices-1ef4f861ce7c)
- [Claude Code Best Practices from Real Projects](https://ranthebuilder.cloud/blog/claude-code-best-practices-lessons-from-real-projects/)
- [Context Engineering Best Practices for AI Dev Teams](https://packmind.com/context-engineering-ai-coding/context-engineering-best-practices/)
- [Cursor AI IDE Setup Guide](https://petronellatech.com/blog/cursor-ai-ide-setup-guide/)

---

*Setup yang baik di awal menghemat waktu berjam-jam di kemudian hari. Jangan skip module ini.*
