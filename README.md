# ⚡ Apagones Mérida — Reportes Ciudadanos

Plataforma de reportes ciudadanos en tiempo real para la ciudad de Mérida, Yucatán. Reporta apagones y fugas de agua directamente en el mapa.

**Live demo:** https://tu-usuario.github.io/apagones-merida/

---

## 🗺️ Funcionalidades

- **Mapa interactivo** de Mérida con secciones electorales coloreadas por tipo de incidencia
- **Dos tipos de reporte**: ⚡ Apagón de luz · 💧 Fuga de agua
- **Expiración automática** a las 3 horas (o cuando alguien reporte que ya se resolvió)
- **Confirmaciones ciudadanas** — otros usuarios pueden confirmar tu reporte
- **Capas independientes** — activa/desactiva cada tipo de incidencia
- **Sin cuenta requerida** — reporte anónimo y rápido

---

## 🚀 Despliegue en GitHub Pages

### 1. Clonar/Fork el repositorio

```bash
git clone https://github.com/tu-usuario/apagones-merida.git
cd apagones-merida
```

### 2. Activar GitHub Pages

1. Ve a **Settings** → **Pages**
2. En *Source*, selecciona **Deploy from a branch**
3. Selecciona la rama `main` y el directorio `/ (root)`
4. Guarda y espera ~2 minutos
5. Tu sitio estará en `https://tu-usuario.github.io/apagones-merida/`

### 3. (Opcional) Dominio personalizado `.mx`

1. Compra el dominio `apagones.mx` en un registrador (Namecheap, Porkbun, etc.)
2. En GitHub Pages → Custom domain, escribe `apagones.mx`
3. En tu registrador, agrega estos registros DNS:
   ```
   A     @     185.199.108.153
   A     @     185.199.109.153
   A     @     185.199.110.153
   A     @     185.199.111.153
   CNAME www   tu-usuario.github.io
   ```
4. Activa **Enforce HTTPS** en GitHub Pages

---

## 🗄️ Actualizar a Supabase (backend real-time)

Por defecto los reportes se guardan en `localStorage` — solo visibles para ti.
Para hacer los reportes **compartidos entre todos los usuarios**:

### 1. Crear proyecto en Supabase (gratis)

Ve a [supabase.com](https://supabase.com) → New Project

### 2. Crear tabla `reports`

```sql
CREATE TABLE reports (
  id          TEXT PRIMARY KEY,
  type        TEXT NOT NULL CHECK (type IN ('apagon', 'agua')),
  lat         DOUBLE PRECISION NOT NULL,
  lng         DOUBLE PRECISION NOT NULL,
  seccion     TEXT DEFAULT '',
  timestamp   BIGINT NOT NULL,
  resolved    BOOLEAN DEFAULT false,
  resolved_at BIGINT,
  confirmations INT DEFAULT 0,
  created_at  TIMESTAMPTZ DEFAULT now()
);

-- Enable Row Level Security
ALTER TABLE reports ENABLE ROW LEVEL SECURITY;

-- Allow anyone to read
CREATE POLICY "Public read" ON reports FOR SELECT USING (true);

-- Allow anyone to insert
CREATE POLICY "Public insert" ON reports FOR INSERT WITH CHECK (true);

-- Allow anyone to update (for confirmations/resolving)
CREATE POLICY "Public update" ON reports FOR UPDATE USING (true);
```

### 3. Agregar Supabase al proyecto

Agrega antes del `</head>` en `index.html`:

```html
<script src="https://cdn.jsdelivr.net/npm/@supabase/supabase-js@2/dist/umd/supabase.min.js"></script>
```

Luego reemplaza las funciones de storage en `index.html`:

```javascript
// Al inicio del script
const SUPABASE_URL = 'https://xxxxxxxx.supabase.co';
const SUPABASE_KEY = 'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...'; // anon key
const supabase = window.supabase.createClient(SUPABASE_URL, SUPABASE_KEY);

// Reemplazar loadReports()
async function loadReports() {
  const cutoff = Date.now() - REPORT_TTL_MS;
  const { data } = await supabase
    .from('reports')
    .select('*')
    .eq('resolved', false)
    .gt('timestamp', cutoff);
  return data || [];
}

// Reemplazar addReport()
async function addReport(report) {
  const { data } = await supabase.from('reports').insert([report]).select();
  return data?.[0];
}

// Reemplazar resolveReport()
async function resolveReport(id) {
  await supabase.from('reports')
    .update({ resolved: true, resolved_at: Date.now() })
    .eq('id', id);
}

// Reemplazar confirmReport()
async function confirmReport(id) {
  await supabase.rpc('increment_confirmations', { report_id: id });
}
```

### 4. Real-time subscriptions (actualizaciones al instante)

```javascript
// Agregar al final del init
supabase.channel('reports').on(
  'postgres_changes',
  { event: '*', schema: 'public', table: 'reports' },
  () => renderMarkers()
).subscribe();
```

---

## 📁 Estructura del proyecto

```
apagones-merida/
├── index.html          # Aplicación completa (single-page)
└── README.md
```

---

## 🔧 Datos del mapa

Los polígonos de distritos se cargan automáticamente desde:  
`https://www.google.com/maps/d/u/0/kml?forcekml=1&mid=10ZjeDwOsMnrbiXpU2NSUh3TJUJbvGdw`

Esto usa un CORS proxy (`api.allorigins.win`) para cargar el KML en el browser.

**Para producción**, descarga el KML y guárdalo en el repo:
```bash
curl -L "https://www.google.com/maps/d/u/0/kml?forcekml=1&mid=10ZjeDwOsMnrbiXpU2NSUh3TJUJbvGdw" \
  -o data/distritos.kml
```
Y en `index.html` cambia el `KML_URL` a `'data/distritos.kml'`.

---

## 📜 Licencia

MIT — Libre para uso, modificación y distribución.
Hecho con ❤️ para Mérida, Yucatán.
