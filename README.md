# Supervisa – Supervisión inteligente de servicios en campo

Plataforma operativa integral: React 19 + TypeScript + Vite + Tailwind CSS + Node.js + Express + MySQL + Socket.IO.

## Puesta en marcha
```bash
npm install                     # Dependencias backend
npm --prefix frontend install   # Dependencias del frontend React (si se instala por primera vez)
npm run build                   # Compila la app React a public/
npm start                       # Inicia el servidor en http://localhost:3000
```

### URLs de acceso
- **Panel Web del Coordinador (`/`)**: Interfaz completa de control operativo, dashboard, mapa en vivo, visitas, novedades, programación y reportes.
- **App Móvil del Supervisor (`/supervisor.html`)**: PWA offline-first con IndexedDB, GPS, escáner QR de centros de costo y captura de evidencias fotográficas.

### Credenciales de acceso
- **Coordinador operativo (conectado a MySQL)**: `coordinador@demo.com` · clave `123456`
- **Coordinador demostración Figma**: `sofia@supervisa.co` · clave `Demo2026!`
- **Supervisores de campo**: `ana@demo.com` / `123456`, `luis@demo.com` / `123456`

## Endpoints
| Método | Ruta | Rol | Para qué |
|---|---|---|---|
| POST | /api/auth/login | — | Devuelve JWT (7 días) |
| GET | /api/mis-asignaciones | supervisor | Centros + checklist para cachear offline |
| POST | /api/visitas | supervisor | Sincroniza una visita (idempotente por UUID) |
| POST | /api/evidencias | supervisor | Sube una foto (multipart, campo `foto`) |
| GET | /api/visitas, /api/visitas/:id | coordinador | Historial y detalle |
| GET/PATCH | /api/novedades | coordinador | Listar y cerrar novedades |
| GET | /api/dashboard | coordinador | Resumen del día |
| GET | /api/centros/:id/qr | coordinador | Payload firmado para imprimir el QR |
| GET | /api/health | — | Detectar conexión real |

Socket.IO: el coordinador se conecta con `auth: { token }` y recibe `novedad:nueva`.

## Cuerpo de POST /api/visitas
```json
{
  "id": "uuid-generado-en-el-cliente",
  "centroCostoId": 1,
  "qr": "1.abc123...",
  "enviadoTs": 1767225600000,
  "checkIn":  { "ts": 1767222000000, "lat": 10.996, "lng": -74.802, "precision": 18 },
  "checkOut": { "ts": 1767224400000, "lat": 10.996, "lng": -74.802 },
  "observaciones": "Todo en orden",
  "respuestas": [{ "actividadId": 1, "cumplida": true, "observacion": "" }],
  "novedades": [{ "id": "uuid", "descripcion": "Fuga en baño piso 3", "prioridad": "alta", "ts": 1767223000000 }]
}
```

## Puntaje de confianza (se calcula en el servidor)
Parte de 100 y resta: fuera de geocerca −40, sin GPS −40, QR inválido −25, check-in en el futuro −20,
desfase de reloj −15, precisión GPS baja −10, visita muy corta −10.
Menos de 70 → la visita queda `en_revision` para el coordinador.
Los umbrales se ajustan en `.env`.

## Orden de sincronización que debe respetar la PWA
1. `POST /api/visitas` (datos, checklist y novedades)  2. `POST /api/evidencias` por cada foto.
Si una foto llega antes que su visita, responde 409 y el cliente reintenta.
