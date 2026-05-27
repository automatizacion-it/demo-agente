# demo-agente — Resumen completo del proyecto

## Arquitectura final

```
Cada 5 minutos
        ↓
health-check.yml  →  ubuntu runner (2vCPU / 7GB)
        ↓
log-sheets.yml    →  escribe en Google Sheets
        ↓
deploy-pages.yml  →  publica en GitHub Pages
        ↓
index.html        →  dashboard en tiempo real
```

---

## Paso 1 — Crear repo local

```powershell
mkdir demo-agente && cd demo-agente
git init
echo "# Demo Agente" > README.md
git add . && git commit -m "first commit"
```

---

## Paso 2 — Crear repo en GitHub

```powershell
gh repo create demo-agente --public --source=. --remote=origin --push
```

---

## Paso 3 — Crear workflow health-check (PowerShell)

```powershell
New-Item -ItemType Directory -Force -Path .github\workflows

@'
name: Health Check cada 5 minutos

on:
  schedule:
    - cron: '*/5 * * * *'
  workflow_dispatch:

jobs:
  verificar:
    runs-on: ubuntu-latest
    permissions:
      issues: write
    steps:
      - name: Verificar estado del repo
        id: check
        run: |
          echo "Verificacion ejecutada: $(date -u)"
          echo "Rama: ${{ github.ref }}"
          echo "Repo: ${{ github.repository }}"
      - name: Crear issue si hay fallo
        if: failure()
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          gh issue create \
            --title "Fallo en health check - $(date -u)" \
            --body "El workflow fallo. Revisar logs en Actions."
'@ | Out-File -FilePath .github\workflows\health-check.yml -Encoding utf8
```

---

## Paso 4 — Crear workflow deploy a GitHub Pages

```powershell
@'
name: Deploy a GitHub Pages

on:
  push:
    branches:
      - main
  workflow_dispatch:

permissions:
  contents: write

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Descargar codigo del repo
        uses: actions/checkout@v4
      - name: Publicar en GitHub Pages
        uses: peaceiris/actions-gh-pages@v4
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: ./
          publish_branch: gh-pages
'@ | Out-File -FilePath .github\workflows\deploy-pages.yml -Encoding utf8
```

---

## Paso 5 — Crear workflow log a Google Sheets

```powershell
@'
name: Log a Google Sheets

on:
  workflow_run:
    workflows: ["Health Check cada 5 minutos"]
    types:
      - completed
  workflow_dispatch:

jobs:
  log-sheets:
    runs-on: ubuntu-latest
    steps:
      - name: Descargar repo
        uses: actions/checkout@v4
      - name: Instalar dependencias
        run: pip install gspread google-auth
      - name: Escribir en Google Sheets
        env:
          GOOGLE_CREDENTIALS: ${{ secrets.GOOGLE_CREDENTIALS }}
          SHEET_ID: 1cjav60NnPKarLQYFi2NWGL2AnVg1kgVqyhxaRAo1Y-A
          RUN_STATUS: ${{ github.event.workflow_run.conclusion || 'manual' }}
          RUN_ID: ${{ github.event.workflow_run.id || github.run_id }}
          RUN_BRANCH: ${{ github.event.workflow_run.head_branch || 'main' }}
          RUN_EVENT: ${{ github.event.workflow_run.event || 'workflow_dispatch' }}
          RUN_DURATION: ${{ github.event.workflow_run.run_duration_ms || '0' }}
        run: |
          cat << 'EOF' > log_run.py
          import gspread, json, os
          from google.oauth2.service_account import Credentials
          from datetime import datetime

          creds_json = json.loads(os.environ['GOOGLE_CREDENTIALS'])
          scopes = ['https://www.googleapis.com/auth/spreadsheets']
          creds = Credentials.from_service_account_info(creds_json, scopes=scopes)
          client = gspread.authorize(creds)
          sheet = client.open_by_key(os.environ['SHEET_ID']).sheet1
          now = datetime.utcnow()
          duration_s = int(os.environ.get('RUN_DURATION', 0)) // 1000
          row = [
            now.strftime('%Y-%m-%d'),
            now.strftime('%H:%M:%S'),
            os.environ.get('RUN_STATUS', 'unknown'),
            str(duration_s) + 's',
            os.environ.get('RUN_BRANCH', 'main'),
            os.environ.get('RUN_ID', ''),
            os.environ.get('RUN_EVENT', ''),
          ]
          sheet.append_row(row)
          print(f"Fila escrita: {row}")
          EOF
          python log_run.py
'@ | Out-File -FilePath .github\workflows\log-sheets.yml -Encoding utf8
```

---

## Paso 6 — Subir archivos al repo

```powershell
git add .
git commit -m "feat: workflows completos"
git push origin main
```

---

## Paso 7 — Activar GitHub Pages

```powershell
[System.IO.File]::WriteAllText("$PWD\pages.json", '{"source":{"branch":"gh-pages","path":"/"}}')

gh api repos/automatizacion-it/demo-agente/pages `
  --method POST `
  --header "Accept: application/vnd.github+json" `
  --input pages.json

Remove-Item pages.json
```

---

## Paso 8 — Guardar credenciales de Google como secret

```powershell
Get-Content "C:\descargas\active-pillar-497602-r4-0a87f0500797.json" -Raw | `
  gh secret set GOOGLE_CREDENTIALS --repo automatizacion-it/demo-agente
```

---

## Paso 9 — Verificar todo

```powershell
# Ver workflows activos
gh workflow list

# Ver últimos runs
gh run list

# Ver secrets
gh secret list --repo automatizacion-it/demo-agente

# Ejecutar manualmente
gh workflow run health-check.yml
gh workflow run log-sheets.yml

# Abrir dashboard
Start-Process "https://automatizacion-it.github.io/demo-agente/"

# Abrir Google Sheets
Start-Process "https://docs.google.com/spreadsheets/d/1cjav60NnPKarLQYFi2NWGL2AnVg1kgVqyhxaRAo1Y-A/edit"
```

---

## Resultado final

| Componente | URL |
|---|---|
| Repositorio | https://github.com/automatizacion-it/demo-agente |
| Dashboard | https://automatizacion-it.github.io/demo-agente/ |
| Google Sheets | https://docs.google.com/spreadsheets/d/1cjav60NnPKarLQYFi2NWGL2AnVg1kgVqyhxaRAo1Y-A |
| Actions | https://github.com/automatizacion-it/demo-agente/actions |

---

## Conceptos aprendidos

| Concepto | Descripción |
|---|---|
| **GitHub Actions** | Automatización de tareas en la nube |
| **Runner** | VM ubuntu efímera 2vCPU / 7GB RAM |
| **Workflow** | Archivo YAML que define los pasos |
| **Secret** | Variable cifrada almacenada en GitHub |
| **GitHub Pages** | Hosting gratuito de sitios estáticos |
| **Service Account** | Cuenta de servicio para APIs de Google |
| **CI/CD** | Integración y entrega continua |
| **Infraestructura efímera** | VM que nace, trabaja y muere sola |

---

> Proyecto creado el 26/05/2026  
> automatizacion-it · demo-agente
