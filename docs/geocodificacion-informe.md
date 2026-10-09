# Informe Técnico: Geocodificación en GeoLibre

> **Proyecto:** GeoLibre — visor GIS web open source  
> **Fecha:** 2026-08-05  
> **Contexto:** Integración de búsqueda de direcciones y geocodificación inversa para un caso de uso en España (ambientóloga/GIS, migrando desde ArcGIS Online)

---

## 1. Resumen ejecutivo

Se han probado en vivo tres APIs de geocodificación aplicables a España. La recomendación es un **plugin de GeoLibre** que usa **CartoCiudad** como proveedor principal (datos oficiales del IGN/Catastro/Correos, sin auth, con CORS habilitado) y **Nominatim/Photon** como fallback internacional. La geocodificación inversa sigue el mismo patrón. No es necesario self-hostear en una primera fase.

---

## 2. APIs investigadas (probadas en vivo, 2026-08-05)

### 2.1 CartoCiudad (IGN/CNIG) — ⭐ Recomendada para España

URL base: `https://www.cartociudad.es/geocoder/api/geocoder/`

| Endpoint | Método | Parámetros | Estado |
|----------|--------|-----------|--------|
| `/find` | GET | `q` (texto libre) | ✅ **Funciona** |
| `/reverseGeocode` | GET | `lat`, `lon` | ✅ **Funciona** |
| `/reverse.json` | — | — | ❌ 404 (no existe) |
| `/search`, `/autocomplete`, `/suggest` | — | — | ❌ 404 (no existen) |

**Características verificadas:**
- Sin autenticación, sin API key
- CORS habilitado (`Access-Control-Allow-Origin: *`) → **llamada directa desde el navegador**
- Devuelve **referencia catastral**, código postal, número de portal, tipo de vía
- Geocodificación a nivel de **portal** (exactitud máxima en España)
- Sin límite de uso documentado (servicio público oficial)

**Respuesta de `find` (ejemplo real, "Calle Mayor 1 Madrid"):**
```json
{
  "id": "13.PV.MUN_280790145199",
  "province": "Madrid",
  "provinceCode": "28",
  "comunidadAutonoma": "Comunidad de Madrid",
  "comunidadAutonomaCode": "13",
  "muni": "Madrid",
  "muniCode": "28079",
  "type": "portal",
  "address": "MAYOR",
  "postalCode": "28013",
  "poblacion": "Madrid",
  "tip_via": "CALLE",
  "lat": 40.416461059323886,
  "lng": -3.7046584169537073,
  "portalNumber": 1,
  "refCatastral": "0343302VK4704C",
  "countryCode": "011"
}
```

**Respuesta de `reverseGeocode` (ejemplo real, 40.4168, -3.7038):**
```json
{
  "id": "13.PV.MUN_280790842318",
  "province": "Madrid",
  "type": "portal",
  "address": "PUERTA DEL SOL",
  "tip_via": "PLAZA",
  "postalCode": "28012",
  "lat": 40.41652681555059,
  "lng": -3.7037876690807514,
  "portalNumber": 7,
  "refCatastral": "0444101VK4704C"
}
```

**Limitaciones detectadas:**
- `find` devuelve un único resultado (geocodificación determinista, no búsqueda difusa con sugerencias)
- No hay endpoint de autocompletado → implementar debounce en el cliente
- Sin documentación oficial activa en la web (los PDFs antiguos referencian endpoints que han cambiado)

---

### 2.2 Nominatim (OpenStreetMap) — Fallback internacional

URL base: `https://nominatim.openstreetmap.org/`

| Endpoint | Método | Parámetros | Estado |
|----------|--------|-----------|--------|
| `/search` | GET | `q`, `format`, `addressdetails`, `limit` | ✅ Funciona |
| `/reverse` | GET | `lat`, `lon`, `format`, `addressdetails` | ✅ Funciona |

**Características:**
- Gratis, sin API key
- Requiere `User-Agent` header identificativo
- **Límite de uso: 1 petición/segundo** (política de uso aceptable de OSM)
- Devuelve múltiples resultados con `addressdetails=1`
- CORS habilitado
- Cobertura global basada en OpenStreetMap

**⚠️ Importante:** El límite de 1 req/seg lo hace inadecuado para autocompletado en tiempo real sin caché ni proxy intermedio.

---

### 2.3 Photon (komoot) — Alternativa self-hostable

URL pública: `https://photon.komoot.io/`

| Endpoint | Método | Parámetros | Estado |
|----------|--------|-----------|--------|
| `/api/` | GET | `q`, `limit`, `lat`, `lon` (priorización) | ✅ Funciona |
| `/reverse` | GET | `lat`, `lon` | ✅ Funciona |

**Características:**
- Basado en OpenStreetMap, construido sobre Elasticsearch
- **Devuelve GeoJSON FeatureCollection** (formato nativo para GeoLibre/MapLibre)
- Soporta autocompletado progresivo (diseñado para type-ahead)
- **Self-hostable** con Docker (`docker run -p 2322:3000 komoot/photon`)
- Sin límites de uso documentados en la instancia pública (pero se recomienda self-host para producción)

**Respuesta de `/api/` (formato GeoJSON):**
```json
{
  "type": "FeatureCollection",
  "features": [{
    "type": "Feature",
    "properties": {
      "name": "Calle Mayor",
      "city": "Madrid",
      "state": "Comunidad de Madrid",
      "postcode": "28013",
      "country": "España",
      "countrycode": "ES"
    },
    "geometry": {
      "type": "Point",
      "coordinates": [-3.7135119, 40.4150583]
    }
  }]
}
```

---

### 2.4 Pelias / Geocode.earth

- `https://api.geocode.earth/v1/search` → **401 Unauthorized** (requiere API key de pago)
- Pelias es self-hostable (Docker), pero requiere infraestructura (Elasticsearch + PostGIS + importadores de OSM)
- **No recomendado** para GeoLibre a menos que se necesite self-hosting completo con datos custom

---

### 2.5 PostGIS + tiger-geocoder

- `tiger_geocoder` es la extensión oficial de PostGIS para geocodificación
- Diseñada para datos del **TIGER/Line del US Census** → **no sirve para España**
- Para España con PostGIS se necesitaría importar los datos de Callejero CartoCiudad (descarga gratuita en formato SHAPE) y construir un geocodificador custom con `pg_trgm` / `tsvector`
- **Evaluación:** viable solo para despliegues con requisitos estrictos de self-hosting y volumen masivo de geocodificación batch. Sobrecarga de mantenimiento alta.

---

## 3. Cuadro comparativo

| Criterio | CartoCiudad | Nominatim | Photon | Pelias self-host | PostGIS/tiger |
|----------|-------------|-----------|--------|------------------|---------------|
| **Auth** | Ninguna | User-Agent | Ninguna | — | — |
| **CORS** | ✅ Sí | ✅ Sí | ✅ Sí | Configurable | N/A (backend) |
| **Cobertura ES** | ⭐⭐⭐ Oficial | ⭐⭐ OSM | ⭐⭐ OSM | ⭐⭐ OSM | ⭐⭐⭐ (si importas CartoCiudad) |
| **Nivel de portal** | ✅ Exacto | ⚠️ Variable | ⚠️ Calle | ⚠️ Variable | Según datos |
| **Ref. catastral** | ✅ Sí | ❌ No | ❌ No | ❌ No | ✅ (si importas) |
| **Código postal** | ✅ Sí | ✅ Sí | ✅ Sí | ✅ Sí | ✅ (si importas) |
| **Autocompletado** | ❌ No | ⚠️ Limitado | ✅ Sí | ✅ Sí | Custom |
| **Límites** | Ninguno | 1 req/seg | Instancia pública limitada | Los que tú pongas | N/A |
| **Self-host** | ❌ No | ❌ No | ✅ Docker | ✅ Docker | ✅ |
| **Coste** | Gratis | Gratis | Gratis/Self-host | Self-host | Self-host |

---

## 4. Arquitectura recomendada

### 4.1 Decisión: Plugin de GeoLibre (cliente directo, sin backend)

```
┌─────────────────────────────────────────────┐
│              GeoLibre (browser)              │
│  ┌───────────────────────────────────────┐  │
│  │  Plugin: geolibre-geocoder             │  │
│  │  ┌─────────────┐  ┌────────────────┐  │  │
│  │  │ Search box  │  │ Click-to-      │  │  │
│  │  │ (debounce)  │  │ reverse        │  │  │
│  │  └──────┬──────┘  └───────┬────────┘  │  │
│  │         │                  │           │  │
│  │  ┌──────▼──────────────────▼───────┐  │  │
│  │  │   GeocoderService (TS)          │  │  │
│  │  │   - cartociudadGeocoder.ts      │  │  │
│  │  │   - nominatimGeocoder.ts        │  │  │
│  │  │   - unified interface           │  │  │
│  │  └──────┬──────────────────────────┘  │  │
│  └─────────┼─────────────────────────────┘  │
└────────────┼────────────────────────────────┘
             │ fetch() directo (CORS ok)
     ┌───────▼───────┐
     │  CartoCiudad   │  ← Principal (ES)
     │  IGN/CNIG API  │
     └───────────────┘
     ┌───────────────┐
     │   Nominatim    │  ← Fallback
     │   (OSM)        │
     └───────────────┘
```

**Razones para cliente directo (no backend):**

1. **CORS habilitado** en CartoCiudad y Nominatim → `fetch()` directo funciona
2. **Sin API keys** que ocultar → no hay razón de seguridad para un proxy
3. **Latencia menor** → sin hop intermedio
4. **Simplicidad** → un plugin autocontenido, sin dependencia de geo-gateway
5. **Funciona en desktop (Tauri) y web** sin cambios

**Razones para añadir un backend (solo si...) y cuándo:**

- Self-hosting de Photon → el backend corre Photon y expone un endpoint propio
- Rate-limiting / caché centralizado → si hay muchos usuarios concurrentes
- Privacy proxy → para no exponer la IP de los usuarios a terceros
- En ese caso, el backend sería **geo-gateway** (FastAPI, ya existe), añadiendo un router `/geocode`

### 4.2 Patrón de integración en GeoLibre

GeoLibre expone tres superficies de UI para plugins:

| Superficie | API | Uso recomendado |
|------------|-----|-----------------|
| **Floating Panel** | `registerFloatingPanel` | Caja de búsqueda sobre el mapa (estilo Google Maps) |
| **Map Control** | `addMapControl` | Botón compacto en esquina del mapa |
| **Toolbar Menu** | `registerToolbarMenu` | Acceso desde barra superior |

**Recomendación:** Floating Panel para la caja de búsqueda + Map click handler para geocodificación inversa (click en mapa → mostrar dirección).

### 4.3 CSP para Tauri (importante)

El desktop build usa Tauri con CSP estricto. **Hay que añadir los dominios de geocodificación al allowlist** en `src-tauri/tauri.conf.json`:

```json
{
  "app": {
    "security": {
      "csp": "... connect-src 'self' https://www.cartociudad.es https://nominatim.openstreetmap.org https://photon.komoot.io ..."
    }
  }
}
```

Sin esto, las peticiones fallan silenciosamente en el desktop.

---

## 5. Ejemplos de código

### 5.1 Servicio de geocodificación unificado (TypeScript)

```typescript
// src/services/geocoder.ts

export interface GeocodeResult {
  lat: number;
  lng: number;
  label: string;          // "Calle Mayor 1, 28013 Madrid"
  type: string;           // "portal" | "street" | "poblacion"
  postalCode?: string;
  province?: string;
  municipality?: string;
  refCatastral?: string;  // Solo CartoCiudad
  source: "cartociudad" | "nominatim" | "photon";
  raw?: unknown;          // Respuesta original
}

export interface ReverseGeocodeResult extends GeocodeResult {
  distance?: number; // metros desde el punto consultado
}

/**
 * Interfaz unificada que implementan todos los proveedores.
 */
export interface GeocoderProvider {
  id: string;
  geocode(query: string): Promise<GeocodeResult[]>;
  reverse(lat: number, lng: number): Promise<ReverseGeocodeResult | null>;
}

// ─── CartoCiudad ───────────────────────────────────────────

const CARTOCIUDAD_BASE = "https://www.cartociudad.es/geocoder/api/geocoder";

export class CartoCiudadGeocoder implements GeocoderProvider {
  id = "cartociudad" as const;

  async geocode(query: string): Promise<GeocodeResult[]> {
    if (!query.trim()) return [];
    const url = `${CARTOCIUDAD_BASE}/find?q=${encodeURIComponent(query)}`;
    const res = await fetch(url);
    if (!res.ok) throw new Error(`CartoCiudad find: ${res.status}`);
    const data = await res.json();

    // find devuelve un objeto único, no un array
    if (!data || !data.lat) return [];
    return [this.mapResult(data)];
  }

  async reverse(lat: number, lng: number): Promise<ReverseGeocodeResult | null> {
    const url = `${CARTOCIUDAD_BASE}/reverseGeocode?lat=${lat}&lon=${lng}`;
    const res = await fetch(url);
    if (!res.ok) return null;
    const data = await res.json();
    if (!data || !data.lat) return null;
    return this.mapResult(data);
  }

  private mapResult(d: Record<string, unknown>): GeocodeResult {
    const tipVia = (d.tip_via as string) || "";
    const address = (d.address as string) || "";
    const portal = d.portalNumber ? ` ${(d.portalNumber as number)}` : "";
    const label = `${tipVia} ${address}${portal}, ${d.postalCode ?? ""} ${d.muni ?? ""}`.trim();
    return {
      lat: d.lat as number,
      lng: d.lng as number,
      label,
      type: (d.type as string) ?? "unknown",
      postalCode: (d.postalCode as string) ?? undefined,
      province: (d.province as string) ?? undefined,
      municipality: (d.muni as string) ?? undefined,
      refCatastral: (d.refCatastral as string) ?? undefined,
      source: "cartociudad",
      raw: d,
    };
  }
}

// ─── Nominatim ─────────────────────────────────────────────

const NOMINATIM_BASE = "https://nominatim.openstreetmap.org";

export class NominatimGeocoder implements GeocoderProvider {
  id = "nominatim" as const;

  async geocode(query: string): Promise<GeocodeResult[]> {
    if (!query.trim()) return [];
    const params = new URLSearchParams({
      q: query,
      format: "json",
      addressdetails: "1",
      limit: "5",
      // Priorizar España
      countrycodes: "es",
    });
    const res = await fetch(`${NOMINATIM_BASE}/search?${params}`, {
      headers: { "User-Agent": "GeoLibre/1.0" },
    });
    if (!res.ok) throw new Error(`Nominatim search: ${res.status}`);
    const data = (await res.json()) as Record<string, unknown>[];
    return data.map((d) => this.mapResult(d));
  }

  async reverse(lat: number, lng: number): Promise<ReverseGeocodeResult | null> {
    const params = new URLSearchParams({
      lat: String(lat),
      lon: String(lng),
      format: "json",
      addressdetails: "1",
    });
    const res = await fetch(`${NOMINATIM_BASE}/reverse?${params}`, {
      headers: { "User-Agent": "GeoLibre/1.0" },
    });
    if (!res.ok) return null;
    const d = await res.json();
    if (!d || !d.lat) return null;
    return this.mapResult(d);
  }

  private mapResult(d: Record<string, unknown>): GeocodeResult {
    const addr = (d.address as Record<string, string>) ?? {};
    const road = addr.road ?? "";
    const num = addr.house_number ?? "";
    const label =
      (d.display_name as string) ??
      `${num} ${road}, ${addr.postcode ?? ""} ${addr.city ?? ""}`;
    return {
      lat: parseFloat(d.lat as string),
      lng: parseFloat(d.lon as string),
      label,
      type: (d.type as string) ?? "unknown",
      postalCode: addr.postcode,
      province: addr.state,
      municipality: addr.city ?? addr.town ?? addr.village,
      source: "nominatim",
      raw: d,
    };
  }
}

// ─── Servicio unificado con fallback ──────────────────────

export class GeocoderService {
  private providers: GeocoderProvider[];

  constructor(providers?: GeocoderProvider[]) {
    this.providers = providers ?? [
      new CartoCiudadGeocoder(), // Principal: datos oficiales ES
      new NominatimGeocoder(),   // Fallback: cobertura global
    ];
  }

  async geocode(query: string): Promise<GeocodeResult[]> {
    const results: GeocodeResult[] = [];
    for (const p of this.providers) {
      try {
        const r = await p.geocode(query);
        results.push(...r);
        // Si el primer proveedor ya tiene resultado, no continuamos
        if (results.length > 0) break;
      } catch (e) {
        console.warn(`Geocoder ${p.id} failed:`, e);
      }
    }
    return results;
  }

  async reverse(lat: number, lng: number): Promise<ReverseGeocodeResult | null> {
    for (const p of this.providers) {
      try {
        const r = await p.reverse(lat, lng);
        if (r) return r;
      } catch (e) {
        console.warn(`Reverse ${p.id} failed:`, e);
      }
    }
    return null;
  }
}
```

### 5.2 Plugin de GeoLibre (TypeScript)

```typescript
// src/geocoder-plugin.ts
// Plugin que registra un panel flotante con búsqueda + click-to-reverse

import type {
  GeoLibreAppAPI,
  GeoLibreFloatingPanelRegistration,
  GeoLibrePlugin,
} from "@geolibre/plugins";
import { GeocoderService, type GeocodeResult } from "./services/geocoder";

const geocoder = new GeocoderService();
let cleanupReverseClick: (() => void) | null = null;

function renderSearchPanel(container: HTMLElement, app: GeoLibreAppAPI): () => void {
  container.innerHTML = `
    <div id="gl-geocoder" style="padding:12px; font-family: sans-serif;">
      <input
        type="text"
        id="gl-geocoder-input"
        placeholder="Buscar dirección en España..."
        style="width: 100%; padding: 8px; border: 1px solid #ccc; border-radius: 4px; box-sizing: border-box;"
        autocomplete="off"
      />
      <ul id="gl-geocoder-results"
          style="list-style: none; padding: 0; margin: 8px 0 0; max-height: 300px; overflow-y: auto;">
      </ul>
    </div>
  `;

  const input = container.querySelector("#gl-geocoder-input") as HTMLInputElement;
  const resultsList = container.querySelector("#gl-geocoder-results") as HTMLUListElement;

  let debounceTimer: ReturnType<typeof setTimeout> | null = null;
  let abortController: AbortController | null = null;

  async function performSearch(query: string) {
    if (abortController) abortController.abort();
    abortController = new AbortController();

    resultsList.innerHTML = '<li style="padding:4px;color:#888;">Buscando...</li>';
    try {
      const results: GeocodeResult[] = await geocoder.geocode(query);
      renderResults(results);
    } catch {
      resultsList.innerHTML = '<li style="padding:4px;color:#c00;">Error en la búsqueda</li>';
    }
  }

  function renderResults(results: GeocodeResult[]) {
    if (results.length === 0) {
      resultsList.innerHTML = '<li style="padding:4px;color:#888;">Sin resultados</li>';
      return;
    }
    resultsList.innerHTML = results
      .map(
        (r, i) => `
        <li data-idx="${i}" style="padding:8px;border-bottom:1px solid #eee;cursor:pointer;">
          <div style="font-size:13px;">${r.label}</div>
          <div style="font-size:11px;color:#888;">
            ${r.source} · ${r.type}${r.refCatastral ? " · RC: " + r.refCatastral : ""}
          </div>
        </li>`
      )
      .join("");

    resultsList.querySelectorAll("li[data-idx]").forEach((li) => {
      li.addEventListener("click", () => {
        const idx = parseInt(li.getAttribute("data-idx")!);
        const result = results[idx];
        flyToResult(app, result);
      });
    });
  }

  function flyToResult(app: GeoLibreAppAPI, r: GeocodeResult) {
    app.fitBounds?.([r.lng - 0.001, r.lat - 0.001, r.lng + 0.001, r.lat + 0.001]);
  }

  // Debounce de 400ms (CartoCiudad no tiene límite, pero evita spam)
  input.addEventListener("input", () => {
    const q = input.value.trim();
    if (q.length < 3) {
      resultsList.innerHTML = "";
      return;
    }
    if (debounceTimer) clearTimeout(debounceTimer);
    debounceTimer = setTimeout(() => performSearch(q), 400);
  });

  return () => {
    if (debounceTimer) clearTimeout(debounceTimer);
    if (abortController) abortController.abort();
  };
}

function setupReverseGeocoding(app: GeoLibreAppAPI): () => void {
  const map = app.getMap?.();
  if (!map) return () => {};

  let popup: { remove: () => void } | null = null;

  // Shift+Click en el mapa → geocodificación inversa
  const onClick = async (e: { lngLat: { lat: number; lng: number }; originalEvent: MouseEvent }) => {
    if (!e.originalEvent.shiftKey) return;

    const { lat, lng } = e.lngLat;

    // Popup de "cargando"
    const maplibre = await import("maplibre-gl");
    popup = new maplibre.Popup()
      .setLngLat([lng, lat])
      .setHTML('<div style="padding:8px;">Consultando dirección...</div>')
      .addTo(map);

    try {
      const result = await geocoder.reverse(lat, lng);
      if (result) {
        popup.setHTML(
          `<div style="padding:8px;min-width:200px;">
            <strong>${result.label}</strong><br/>
            <small>${result.source} · ${result.type}</small><br/>
            ${result.postalCode ? "CP: " + result.postalCode + "<br/>" : ""}
            ${result.refCatastral ? "Ref. Catastral: " + result.refCatastral + "<br/>" : ""}
            <small>Coord: ${lat.toFixed(5)}, ${lng.toFixed(5)}</small>
          </div>`
        );
      } else {
        popup.setHTML('<div style="padding:8px;">Sin resultados</div>');
      }
    } catch {
      popup?.setHTML('<div style="padding:8px;">Error</div>');
    }
  };

  map.on("click", onClick);

  return () => {
    map.off("click", onClick);
    popup?.remove();
  };
}

export const geocoderPlugin: GeoLibrePlugin = {
  id: "geolibre-geocoder",
  name: "Geocodificador ES",
  version: "1.0.0",
  activeByDefault: true,

  activate: (app: GeoLibreAppAPI) => {
    // 1. Panel flotante con búsqueda
    const panel: GeoLibreFloatingPanelRegistration = {
      id: "geocoder-panel",
      title: "Buscar dirección",
      icon: "🔍",
      defaultWidth: 340,
      render: (container: HTMLElement) => renderSearchPanel(container, app),
    };
    app.registerFloatingPanel?.(panel);
    app.openFloatingPanel?.("geocoder-panel");

    // 2. Click-to-reverse (Shift+Click)
    cleanupReverseClick = setupReverseGeocoding(app);

    return true;
  },

  deactivate: (app: GeoLibreAppAPI) => {
    app.unregisterFloatingPanel?.("geocoder-panel");
    cleanupReverseClick?.();
    cleanupReverseClick = null;
  },
};
```

### 5.3 Backend opcional en geo-gateway (FastAPI)

Si en el futuro se necesita caché, rate-limiting, o self-hosting de Photon:

```python
# app/geocode/router.py
# Añadir a geo-gateway como router adicional

from fastapi import APIRouter, HTTPException, Query
import httpx
from functools import lru_cache

router = APIRouter(prefix="/geocode", tags=["geocode"])

CARTOCIUDAD_BASE = "https://www.cartociudad.es/geocoder/api/geocoder"
NOMINATIM_BASE = "https://nominatim.openstreetmap.org"

# Caché en memoria simple (sustituir por Redis en producción)
_cache: dict[str, dict] = {}
_cache_ttl = 86400  # 24h


@router.get("/search")
async def search(
    q: str = Query(..., min_length=3, description="Texto a buscar"),
):
    """Geocodificación directa con fallback CartoCiudad → Nominatim."""
    cache_key = f"search:{q.lower().strip()}"
    if cached := _cache.get(cache_key):
        return cached

    async with httpx.AsyncClient(timeout=10) as client:
        # 1. CartoCiudad (principal)
        try:
            r = await client.get(
                f"{CARTOCIUDAD_BASE}/find",
                params={"q": q},
            )
            if r.status_code == 200 and r.json():
                result = _normalize_cartociudad(r.json())
                _cache[cache_key] = result
                return result
        except httpx.HTTPError:
            pass

        # 2. Nominatim (fallback)
        try:
            r = await client.get(
                f"{NOMINATIM_BASE}/search",
                params={
                    "q": q,
                    "format": "json",
                    "addressdetails": 1,
                    "limit": 5,
                    "countrycodes": "es",
                },
                headers={"User-Agent": "GeoLibre-Gateway/1.0"},
            )
            if r.status_code == 200:
                results = [_normalize_nominatim(item) for item in r.json()]
                _cache[cache_key] = {"results": results}
                return {"results": results}
        except httpx.HTTPError:
            pass

    raise HTTPException(status_code=502, detail="Todos los proveedores fallaron")


@router.get("/reverse")
async def reverse(
    lat: float = Query(..., ge=-90, le=90),
    lon: float = Query(..., ge=-180, le=180),
):
    """Geocodificación inversa con fallback."""
    cache_key = f"reverse:{lat:.5f},{lon:.5f}"
    if cached := _cache.get(cache_key):
        return cached

    async with httpx.AsyncClient(timeout=10) as client:
        # 1. CartoCiudad
        try:
            r = await client.get(
                f"{CARTOCIUDAD_BASE}/reverseGeocode",
                params={"lat": lat, "lon": lon},
            )
            if r.status_code == 200 and r.json():
                result = _normalize_cartociudad(r.json())
                _cache[cache_key] = result
                return result
        except httpx.HTTPError:
            pass

        # 2. Nominatim
        try:
            r = await client.get(
                f"{NOMINATIM_BASE}/reverse",
                params={
                    "lat": lat,
                    "lon": lon,
                    "format": "json",
                    "addressdetails": 1,
                },
                headers={"User-Agent": "GeoLibre-Gateway/1.0"},
            )
            if r.status_code == 200 and r.json():
                result = _normalize_nominatim(r.json())
                _cache[cache_key] = result
                return result
        except httpx.HTTPError:
            pass

    raise HTTPException(status_code=502, detail="Geocodificación inversa falló")


def _normalize_cartociudad(d: dict) -> dict:
    tip_via = d.get("tip_via", "")
    address = d.get("address", "")
    portal = d.get("portalNumber", "")
    return {
        "lat": d.get("lat"),
        "lng": d.get("lng"),
        "label": f"{tip_via} {address} {portal}, {d.get('postalCode', '')} {d.get('muni', '')}".strip(),
        "type": d.get("type"),
        "postalCode": d.get("postalCode"),
        "province": d.get("province"),
        "municipality": d.get("muni"),
        "refCatastral": d.get("refCatastral"),
        "source": "cartociudad",
    }


def _normalize_nominatim(d: dict) -> dict:
    addr = d.get("address", {})
    return {
        "lat": float(d.get("lat", 0)),
        "lng": float(d.get("lon", 0)),
        "label": d.get("display_name", ""),
        "type": d.get("type"),
        "postalCode": addr.get("postcode"),
        "province": addr.get("state"),
        "municipality": addr.get("city") or addr.get("town") or addr.get("village"),
        "source": "nominatim",
    }
```

### 5.4 Self-hosting de Photon con Docker (opcional)

```yaml
# docker-compose.photon.yml
# Para máxima privacidad y sin límites de uso

services:
  photon:
    image: ghcr.io/komoot/photon:latest
    ports:
      - "2322:3000"
    volumes:
      - photon-data:/photon/photon_data
    # Primera vez: descargar el dump de ES (aprox. 80 GB descomprimido)
    # docker exec -it <container> java -jar photon.jar -nominatim-import ...

volumes:
  photon-data:
```

Comandos de inicialización:
```bash
# 1. Arrancar el contenedor
docker compose -f docker-compose.photon.yml up -d

# 2. Importar datos de España desde Nominatim (lento, ~80 GB disco)
docker exec -it photon-photon-1 java -jar photon.jar -nominatim-import \
  -host nominatim.example.com -port 5432 -database nominatim -user nominatim \
  -languages es,ca,eu,gl

# 3. Probar
curl "http://localhost:2322/api/?q=Calle+Mayor+Madrid&limit=5"
```

---

## 6. MapLibre GL Geocoder (alternativa con librería existente)

Existe `@maplibre/maplibre-gl-geocoder` que ya integra un control de búsqueda en MapLibre. Sin embargo:

- **No soporta CartoCiudad** como provider nativo (solo OpenStreetMap/Nominatim/Mapbox/Pelias)
- Se puede usar con un **provider custom** implementando la interfaz `GeocoderApi`

```typescript
import MaplibreGeocoder from "@maplibre/maplibre-gl-geocoder";
import "@maplibre/maplibre-gl-geocoder/dist/maplibre-gl-geocoder.css";

// Provider custom para CartoCiudad
const cartoCiudadApi = {
  forwardGeocode: async (config: { query: string }) => {
    const res = await fetch(
      `https://www.cartociudad.es/geocoder/api/geocoder/find?q=${encodeURIComponent(config.query)}`
    );
    const data = await res.json();
    const features = data.lat
      ? [{
          type: "Feature" as const,
          geometry: { type: "Point" as const, coordinates: [data.lng, data.lat] },
          properties: {
            title: `${data.tip_via} ${data.address} ${data.portalNumber}`,
            description: `${data.postalCode} ${data.muni}`,
          },
          place_name: `${data.tip_via} ${data.address} ${data.portalNumber}, ${data.postalCode} ${data.muni}`,
          text: `${data.address} ${data.portalNumber}`,
          center: [data.lng, data.lat],
        }]
      : [];
    return { features };
  },
  reverseGeocode: async (coord: { lat: number; lon: number }) => {
    const res = await fetch(
      `https://www.cartociudad.es/geocoder/api/geocoder/reverseGeocode?lat=${coord.lat}&lon=${coord.lon}`
    );
    const data = await res.json();
    return {
      place_name: data.address ? `${data.tip_via} ${data.address} ${data.portalNumber}` : "",
    };
  },
};

// Uso en GeoLibre
app.addMapControl(
  new MaplibreGeocoder(cartoCiudadApi, {
    maplibregl: await import("maplibre-gl"),
    placeholder: "Buscar dirección...",
    limit: 1,
  }),
  "top-left"
);
```

**Ventaja:** UI pulida (caja de búsqueda con dropdown de sugerencias) lista para usar.
**Desventaja:** Depende del paquete externo; menos control sobre el estilado.

---

## 7. Roadmap de implementación

### Fase 1 — MVP (1-2 días) ✅ Factible ya

1. Crear plugin `geolibre-geocoder` en `packages/plugins/src/plugins/`
2. Implementar `GeocoderService` con CartoCiudad + Nominatim
3. Registrar como Floating Panel
4. Añadir dominios al CSP de Tauri
5. Registrar en `usePlugins.ts`

### Fase 2 — UX mejorada (3-5 días)

1. Usar `@maplibre/maplibre-gl-geocoder` con provider custom (UI profesional)
2. Click en mapa para reverse geocoding (sin Shift)
3. Marcador visual al seleccionar resultado
4. Histórico de búsquedas en localStorage

### Fase 3 — Backend opcional (si se necesita)

1. Router `/geocode` en geo-gateway
2. Caché con Redis (TTL 24h)
3. Rate-limiting por usuario
4. Métricas de uso por proveedor

### Fase 4 — Self-hosting (solo si requisitos de privacidad)

1. Desplegar Photon con Docker
2. Importar datos de España
3. Cambiar `GeocoderService` para apuntar a instancia propia

---

## 8. Pitfalls y notas técnicas

### 8.1 Endpoints de CartoCiudad (verificados 2026-08-05)

| Documentado en foros antiguos | Estado real |
|-------------------------------|-------------|
| `/reverse.json` | ❌ 404 |
| `/reverse?lat=&lon=` | ❌ 404 |
| `/reverseGeocode?lat=&lon=` | ✅ **Correcto** |
| `/search?q=` | ❌ 404 |
| `/autocomplete?q=` | ❌ 404 |
| `/find?q=` | ✅ Correcto (devuelve 1 resultado) |

La documentación oficial está desactualizada en muchos sitios web. Los endpoints verificados son solo `find` y `reverseGeocode`.

### 8.2 CORS

- CartoCiudad: ✅ `Access-Control-Allow-Origin: *`
- Nominatim: ✅ CORS habilitado
- Photon: ✅ CORS habilitado

Todas soportan llamada directa desde el navegador.

### 8.3 Nominatim: User-Agent obligatorio

Nominatim devuelve **403 Forbidden** sin un `User-Agent` identificativo válido. Usar:
```
User-Agent: GeoLibre/1.0 (https://geolibre.app)
```

### 8.4 CartoCiudad no tiene autocompletado

El endpoint `find` devuelve **un único resultado** determinista. Para implementar autocompletado tipo-ahead, hay dos opciones:
- Usar Photon (que sí tiene type-ahead)
- Implementar caché local con historial de búsquedas

### 8.5 Referencias catastrales

Solo CartoCiudad devuelve `refCatastral`. Este es un campo de alto valor para trabajo catastral/ambiental en España y es una razón de peso para usarlo como proveedor principal.

### 8.6 Sistema de coordenadas

Todas las APIs devuelven coordenadas en **WGS84 (EPSG:4326)**, que es lo que usa MapLibre GL. No se necesita reproyección.
