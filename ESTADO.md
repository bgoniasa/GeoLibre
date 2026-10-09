# GeoLibre — Estado del proyecto

> Fuente de verdad. Consultar al empezar cualquier sesión.
> Actualizar después de cada cambio. No fiarse de la memoria del asistente.

**Última verificación:** 2026-08-01 (GitHub API)

## Repo

- **Upstream:** `opengeos/GeoLibre` (NO giswqs)
- **Fork:** `bgoniasa/GeoLibre`
- **Local:** `/opt/data/proyectos/GeoLibre`
- **Rama de trabajo:** `main` (sincronizada con upstream)
- **Upstream remote corregido:** 2026-07-28 → `https://github.com/opengeos/GeoLibre.git`

## Issue #1381: Dashboard enhancements

- **URL:** https://github.com/opengeos/GeoLibre/issues/1381
- **Estado:** Abierto

### Widgets del dashboard

| Widget | Estado | PR | Fecha |
|--------|--------|-----|-------|
| histogram | ✅ En main | — | — |
| scatter | ✅ En main | — | — |
| bar | ✅ En main | — | — |
| line | ✅ En main | — | — |
| box | ✅ En main | — | — |
| pie | ✅ En main | — | — |
| **indicator** | ✅ **Mergeado** | [#1392](https://github.com/opengeos/GeoLibre/pull/1392) | 2026-07-25 |
| **selector + list + cross-filter** | ✅ **MERGEADO** | [#1504](https://github.com/opengeos/GeoLibre/pull/1504) | 2026-08-01 |
| ~~list (separado)~~ | ❌ Cerrado, consolidado en #1504 | ~~#1508~~ | — |
| cross-filtering | ✅ Implementado por giswqs, mergeado en #1504 | — | 2026-08-01 |

### Bugs detectados por claude-review (31-jul)

giswqs mergeó con estos bugs conocidos (probablemente los arregla después):

1. **`normalizeWidgets`** no copia campos del list widget (`listFields`, `sortBy`, `sortDir`, `limit`) → se pierden al guardar
2. **List widget hace shadow** del `rows` memoizado → ignora cross-filtering
3. **Default title del list** usa `layerId` (UUID) en vez de nombre legible

### Siguiente paso

- Los 3 widgets están mergeados. Issue #1381 sigue abierto para más widgets/mejoras.
- Considerar arreglar los 3 bugs del list widget con un follow-up PR.

## Cuenta GitHub

- Usuario: `bgoniasa`
- Credenciales en `.env` (GITHUB_TOKEN, GITHUB_USERNAME, GITHUB_EMAIL)
- **NUNCA** preguntar a Bego por credenciales.

## Cómo verificar estado antes de trabajar

```bash
TOKEN=$(grep '^GITHUB_TOKEN=' /opt/data/.env | cut -d'=' -f2-)
# PRs recientes mergeados
curl -s -H "Authorization: token $TOKEN" \
  "https://api.github.com/repos/opengeos/GeoLibre/pulls?state=closed&sort=updated&direction=desc&per_page=5" | jq '.[] | {number, title, merged_at, user: .user.login}'
# Estado del issue
curl -s -H "Authorization: token $TOKEN" \
  "https://api.github.com/repos/opengeos/GeoLibre/issues/1381" | jq '{state, comments, updated_at}'
```
