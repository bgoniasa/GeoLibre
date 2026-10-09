# DEPLOY GEOLIBRE v3.1 — GeoLens (Easypanel, 27-sep-2026)

Estado del código: fork `bgoniasa/GeoLibre` main = **v3.1.0** (push `32414a84`, verificado).
Plugin de flotas: reconstruido contra v3.1 — build verde, typecheck 0 errores, manifest con `engines:["maplibre"]`.

## Pasos en Easypanel (5 minutos)

1. **Easypanel → tu proyecto → + Servicio → App** (o edita el servicio GeoLens/GeoLibre-web existente si ya lo tienes)
2. **Fuente → Git**: usuario `bgoniasa`, repo `GeoLibre`, rama `main` → Guardar
   - Si pide credenciales: tu PAT de GitHub (scope `repo` — el de siempre)
3. **Compilación → Dockerfile** (el repo trae el suyo, se detecta solo). Deja el puerto interno **80** (nginx del Dockerfile)
4. **Deploy / Implementar** — el build tarda: compila React + JupyterLite con node:22 (~10-15 min la primera vez, cacheado después)
5. **Dominio**: crear/verificar el dominio del servicio → Puerto **80** (solo el número) → Destino Protocolo **HTTP** (¡no HTTPS — la regla de oro de los 404!) → Force SSL ✅

## Variables de entorno del servicio (opciones)

| Variable | Valor | Para qué |
|----------|-------|----------|
| `GEOLIBRE_SHARE_URL` | `https://tu-dominio` (opcional) | Enlaces de compartir |
| `GEOLIBRE_CLERK_PUBLISHABLE_KEY` | (vacío por ahora) | Login gate — FASE 3, todavía no |
| `GEOLIBRE_AUTH0_DOMAIN` / `..._CLIENT_ID` | (vacío) | Ídem |

Con nada: arranca sin login. Perfecto para probar HOY.

## El plugin de flotas (después de que la web viva)

El zip listo: `proyectos/geolibre-flotas/plugin/dist/flotas-demo-0.1.0.zip`
(manifest con `engines:["maplibre"]` verificado dentro).

En GeoLibre v3.1: **Plugins → Cargar/Install plugin → subir zip** → panel "Flotas" aparece a la derecha.
Apuntará al contrato demo (fixtures). Para urbaser/real: `pnpm build:contrato urbaser` y subir ese zip.

## Verificación post-deploy

1. Dominio carga → debe decir **v3.1.0** en About/Settings
2. Cargar un GeoJSON de prueba (drag&drop) → mapa pinta
3. Botón globo 🌐 → Cesium 3D arranca (lo nuevo de 3.1)
4. Plugin flotas zip → panel aparece y lista vehículos demo

## Notas técnicas del merge

- Merge limpio (0 conflictos): nuestro main no tenía cambios propios — TODO lo de Bego ya está integrado en upstream v3.1 (selector #1504, indicator #1392, CartoCiudad #1712 — verificado por autor)
- Las 4 ramas feature viejas siguen intactas en local/origin por si acaso (sus commits ya viven en upstream)
- `core-plugins-types.ts` del plugin actualizado al de v3.1 (1257 líneas) — `registerRightPanel`, `addGeoJsonLayer`, `getMap` siguen existiendo: el plugin compila sin tocar NADA de código
- Si Easypanel se queja de RAM en el build (node+JupyterLite a la vez): pausar Ollama durante el build y reanudar (la RAM del VPS está justa)
