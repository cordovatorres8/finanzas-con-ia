# Arquitectura

## Flujo de un audio

```mermaid
sequenceDiagram
    actor U as Usuario
    participant T as Telegram
    participant W as Cloudflare Worker
    participant A as Apps Script
    participant G as Gemini API
    participant S as Google Sheets
    U->>T: Audio "gasté 45 soles en el almuerzo"
    T->>W: POST webhook (?k=CLAVE)
    W-->>T: 200 OK (inmediato)
    W->>A: POST /exec (en segundo plano, ctx.waitUntil)
    A->>A: Valida clave, usuario y update_id repetido
    A->>T: getFile + descarga del audio
    A->>G: generateContent (audio + instrucciones, salida JSON)
    G-->>A: {"monto":45,"categoria":"Gasto Personal","nota":"Almuerzo","fecha":"2026-10-27"}
    A->>S: appendRow en Movimientos
    A->>A: revisarPresupuesto (80% / 100%)
    A-->>U: Correo de alerta (si corresponde)
    A->>T: sendMessage "✅ Gasto de S/ 45.00 registrado"
```

## Componentes

| Componente | Archivo | Responsabilidad |
|---|---|---|
| Entrada HTTP | `Code.gs` → `doPost`, `doGet` | Telegram, Atajo (JSON con `token`), formulario y dashboard |
| Registro y alertas | `Code.gs` → `registrar`, `revisarPresupuesto`, `avisar` | Guarda el movimiento y envía alertas por tipo de presupuesto |
| Bot | `Telegram.gs` | Webhook, comandos, botones, borrado y llamada a Gemini con reintentos y respaldo de modelos |
| Cuotas | `Cuotas.gs` | Planes de pago, siguiente cuota y reversión al borrar |
| Fórmulas | `Reparar.gs` | Fórmulas de Presupuesto, Resumen y Metas según la configuración regional |
| Dashboard | `Dashboard.html` | Interfaz con HTML, CSS y SVG puros. Se actualiza cada 30 s y solo redibuja si cambian los datos |
| Intermediario | `worker.js` | `200` inmediato a Telegram, `/panel`, `/registro`, `/app` (PWA para Android) y `assetlinks.json` |

## Hojas del Google Sheet

| Hoja | Columnas |
|---|---|
| Movimientos | Fecha · Año-Mes · Categoría · Tipo (Gasto, Ingreso o Ahorro) · Monto · Nota · Registrado |
| Presupuesto | Categoría · Límite · Gastado · % · Estado · Tipo (fijo, servicio o variable) |
| Metas | Categoría · Meta · Fecha límite · Ahorrado acumulado · … |
| Cuotas | Producto · Total · Cuotas · Monto cuota · Pagadas · Fecha compra · Estado · Último pago · Primer pago |

## Decisiones de diseño

- **Google Sheets como base de datos.** Es gratis, el usuario puede ver y editar sus datos, y no hay servidor que mantener.
- **Salida JSON de Gemini** (`responseMimeType: application/json`, `temperature: 0`). Las respuestas son estables y fáciles de validar.
- **Respaldo de modelos dinámico.** Los modelos se retiran con el tiempo, así que el bot consulta `models.list`, prioriza los *flash* y *lite* estables y descarta los que responden 404.
- **El dashboard no usa librerías.** Carga rápido dentro de Apps Script y se controla al detalle, incluidos el modo oscuro y los gráficos SVG.
