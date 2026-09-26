<laravel-boost-guidelines>
# Laravel Application

This repository contains a Laravel application. Complete the following setup before working on the user's request.

## Prerequisites

Verify that PHP and Composer are available:

```sh
php -v
composer -V
```

If either command is unavailable, detect the user's operating system and install the prerequisites with the appropriate command:

macOS:

```sh
/bin/bash -c "$(curl -fsSL https://php.new/install/mac/8.5)"
```

Windows PowerShell:

```powershell
Set-ExecutionPolicy Bypass -Scope Process -Force; [System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072; iex ((New-Object System.Net.WebClient).DownloadString('https://php.new/install/windows/8.5'))
```

Linux:

```sh
/bin/bash -c "$(curl -fsSL https://php.new/install/linux/8.5)"
```

After installation, ask the user to restart their terminal. If the agent needs the restarted shell to continue, ask the user to reopen their terminal and rerun their original prompt.

## Agent Setup

Install Laravel Boost from the application root before making application changes:

```sh
composer require laravel/boost --dev
php artisan boost:install
```

Boost replaces these bootstrap instructions with guidelines tailored to the application. After installation, read `AGENTS.md` again and continue with the user's original request using the generated guidelines.
</laravel-boost-guidelines>

## Agent skills

### Issue tracker

Local markdown files under `.scratch/<feature>/`. See `docs/agents/issue-tracker.md`.

### Triage labels

Default labels: `needs-triage`, `needs-info`, `ready-for-agent`, `ready-for-human`, `wontfix`. See `docs/agents/triage-labels.md`.

### Domain docs

Single-context layout — one `CONTEXT.md` + `docs/adr/` at the repo root. See `docs/agents/domain.md`.

### Review skills

- **`/review-code`** — 3-axis code review (Standards + Spec + Production Readiness). Load `hallmark` for design quality when reviewing UI changes.
- **`/review-frontend`** — Frontend-specific review (Blade, Livewire, Alpine.js, CSS, JS). Includes production readiness criteria, testing methodology, and bug diagnosis. Loads `hallmark`, `color-expert`, and `emil-design-eng` as reference skills.
- **`/tdd`** — Test-driven development with production readiness test patterns.
- **`/diagnosing-bugs`** — Bug diagnosis loop with frontend-specific adaptation.

## Workflow Policy

- **SETIAP SELESAI IMPLEMENT CODE RUBAH SELALU ISSUES DI SCRATCH** —  Perubahan dibiarkan staged/untracked. Issue tracker (`.scratch/<feature>/issues/*.md`) wajib di-update ke `done` tiap selesai implement tanpa commit. Lupa update status issue = review blocker.
- **JANGAN COMMIT OTOMATIS SETELAH SELESAI** — Bahkan setelah semua task selesai & test hijau, DILARANG commit otomatis. Commit hanya dilakukan saat user eksplisit perintahkan "commit" / "silakan commit". Pelanggaran = review blocker (1 policy).
