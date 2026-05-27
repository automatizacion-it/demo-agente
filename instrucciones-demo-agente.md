# Instrucciones — demo-agente GitHub Actions

## Paso 1 — Crear repo local

```bash
mkdir demo-agente && cd demo-agente
git init
echo "# Demo Agente" > README.md
git add . && git commit -m "first commit"
```

## Paso 2 — Crear repo en GitHub

```bash
gh repo create demo-agente --public --source=. --remote=origin --push
```

## Paso 3 — Crear el workflow (PowerShell)

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

Verificar que quedó bien:

```powershell
Get-Content .github\workflows\health-check.yml
```

## Paso 4 — Push del workflow

```bash
git add .github\workflows\health-check.yml
git commit -m "add: health check workflow cada 5 minutos"
git push origin main
```

## Paso 5 — Verificar y ejecutar

```bash
# Ver workflows registrados
gh workflow list

# Ejecutar manualmente (sin esperar 5 min)
gh workflow run health-check.yml

# Ver runs recientes
gh run list --workflow=health-check.yml

# Ver log del último run
gh run view --log

# Crear issue de seguimiento
gh issue create \
  --title "Health Check activado - corre cada 5 minutos" \
  --body "## Workflow activo
- **ID:** 283865867
- **Primer run:** 26484125492
- **Estado:** OK en 9s
- **Trigger:** automatico cada 5 min + manual"
```

---

## Resultado final

| Comando | Resultado |
|---|---|
| `gh repo create` | Repo creado en GitHub |
| `gh workflow list` | `Health Check cada 5 minutos — active` |
| `gh workflow run` | Ejecutado en 9s |
| `gh run list` | Run `26484125492` completado |
| `gh issue create` | Issue #1 abierto en automatizacion-it/demo-agente |

---

> **Nota:** GitHub Actions usa UTC para el cron. El intervalo mínimo es `*/5 * * * *` (cada 5 minutos),
> pero puede haber un pequeño retraso por la cola de runners compartidos.
