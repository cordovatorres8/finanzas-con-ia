# Fragmentos destacados

Tres partes del código que resuelven los problemas más interesantes del proyecto. El sistema completo está en un repositorio privado; si quieres revisarlo (por ejemplo, para una entrevista), escríbeme y te doy acceso.

## 1. Webhook de Telegram sin reintentos (Cloudflare Worker)

Apps Script responde a los webhooks con una redirección 302 y Telegram la toma como error, así que reenvía el mismo mensaje una y otra vez. El Worker responde `200` al instante y pasa el mensaje a Apps Script en segundo plano con `ctx.waitUntil`. Solo acepta llamadas que traen la clave secreta.

```javascript
export default {
  async fetch(request, env, ctx) {
    const url = new URL(request.url);
    // … rutas GET: /panel, /registro, /app, /icon.png, /manifest.json …

    if (request.method !== 'POST') return new Response('ok');
    const k = url.searchParams.get('k');
    if (!env.CLAVE || k !== env.CLAVE) return new Response('ok');   // ignora a cualquiera sin la clave
    const cuerpo = await request.text();
    ctx.waitUntil(
      fetch(env.APPS_SCRIPT_URL + '?k=' + encodeURIComponent(k), {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: cuerpo,
        redirect: 'follow'
      }).catch(() => {})
    );
    return new Response('ok', { status: 200 });
  }
};
```

## 2. Respaldo automático entre modelos de Gemini

Los modelos se saturan (error 503) o Google los retira (error 404). El bot arma solo su lista de modelos disponibles: prueba primero el modelo principal y luego los *lite*, que casi nunca están saturados. Reintenta en dos vueltas y descarta para siempre los modelos retirados.

```javascript
function interpretar(parts) {
  const body = {
    systemInstruction: { parts: [{ text: promptSistema() }] },
    contents: [{ role: 'user', parts: parts }],
    generationConfig: { responseMimeType: 'application/json', temperature: 0 }
  };
  let ultimo = '';
  const malos = {};                                              // modelos que fallaron sin remedio en este mensaje
  const modelos = modelosGemini_();
  for (let vuelta = 1; vuelta <= 2; vuelta++) {                  // 2 vueltas por todos los modelos
  if (vuelta === 2) { log_('Todos saturados, espero 6 s y pruebo otra vez'); Utilities.sleep(6000); }
  for (const modelo of modelos) {
    if (malos[modelo]) continue;
    for (let intento = 1; intento <= 2; intento++) {
      const res = UrlFetchApp.fetch('https://generativelanguage.googleapis.com/v1beta/models/' + modelo + ':generateContent', {
        method: 'post', contentType: 'application/json', muteHttpExceptions: true,
        headers: { 'x-goog-api-key': cfg('GEMINI_API_KEY') }, payload: JSON.stringify(body)
      });
      const code = res.getResponseCode();
      if (code === 200) {
        const txt = JSON.parse(res.getContentText()).candidates[0].content.parts.map(p => p.text || '').join('');
        return JSON.parse(txt.replace(/```json|```/g, '').trim());
      }
      ultimo = modelo + ' ' + code + ': ' + res.getContentText().slice(0, 150);
      log_('Gemini falló (' + ultimo.replace(/\s+/g, ' ') + ') intento ' + intento);
      if (code === 404) { retirarModelo_(modelo); malos[modelo] = true; break; }   // Google lo retiró: no se usa más
      if (code === 400 || code === 403) { malos[modelo] = true; break; }           // no tiene sentido reintentar
      if (intento < 2) Utilities.sleep(1500);                    // 1 reintento rápido y pasa al siguiente modelo
    }
  }
  }
  throw new Error('Gemini (Google) está saturado ahora mismo. No se guardó nada: vuelve a enviar tu mensaje en 1 o 2 minutos. (' + ultimo.slice(0, 120) + ')');
}

function modelosGemini_() {
  const cache = CacheService.getScriptCache();
  let lista = null;
  const enCache = cache.get('gem_modelos');
  if (enCache) lista = JSON.parse(enCache);
  else {
    try {
      const r = UrlFetchApp.fetch('https://generativelanguage.googleapis.com/v1beta/models?pageSize=200', {
        muteHttpExceptions: true, headers: { 'x-goog-api-key': cfg('GEMINI_API_KEY') }
      });
      if (r.getResponseCode() === 200) {
        lista = (JSON.parse(r.getContentText()).models || [])
          .filter(m => (m.supportedGenerationMethods || []).indexOf('generateContent') >= 0)
          .map(m => m.name.replace('models/', ''))
          .filter(n => /^gemini-(\d+(\.\d+)?-)?flash(-lite)?(-latest)?$/.test(n));   // solo flash estables (sin preview/tts/imagen)
        cache.put('gem_modelos', JSON.stringify(lista), 21600);
      } else log_('No pude listar modelos: ' + r.getResponseCode());
    } catch (e) { log_('No pude listar modelos: ' + e); }
  }
  const retirados = JSON.parse(cfg('GEMINI_RETIRADOS') || '[]');
  const ver = n => parseFloat((n.match(/^gemini-(\d+(?:\.\d+)?)/) || [])[1] || 0);
  const lite = n => n.indexOf('-lite') >= 0;
  const auto = (lista || GEMINI_MODELOS).slice().sort((a, b) => (lite(b) - lite(a)) || (ver(b) - ver(a)));
  const orden = [GEMINI_MODEL].concat(auto, GEMINI_MODELOS);
  const out = [];
  orden.forEach(m => { if (m && out.indexOf(m) < 0 && retirados.indexOf(m) < 0) out.push(m); });
  return out.slice(0, 6);
}

function retirarModelo_(modelo) {
  const P = PropertiesService.getScriptProperties();
  const r = JSON.parse(P.getProperty('GEMINI_RETIRADOS') || '[]');
  if (r.indexOf(modelo) < 0) { r.push(modelo); P.setProperty('GEMINI_RETIRADOS', JSON.stringify(r)); }
  log_('Modelo retirado por Google, ya no se usa: ' + modelo);
}
```

## 3. Compras en cuotas sin intereses

El usuario dice *"compré una laptop a 3598 en 12 cuotas"* y Gemini devuelve el total y el número de cuotas. El plan divide el total, y la última cuota absorbe los céntimos para que la suma cuadre exacto.

```javascript
function montoCuota_(p, k) { return k === p.n ? r2_(p.total - p.cuota * (p.n - 1)) : p.cuota; }

function planCuotas_(producto, total, n, montoCuota, primerPago) {
  n = Math.round(Number(n));
  if (!(n >= 2 && n <= 60)) throw new Error('Me faltó el número de cuotas.');
  let cuota = Number(montoCuota) > 0 ? r2_(montoCuota) : (Number(total) > 0 ? r2_(Number(total) / n) : 0);
  if (!(cuota > 0)) throw new Error('Me faltó el precio total o el monto de cada cuota.');
  total = Number(total) > 0 ? r2_(total) : r2_(cuota * n);
  // Mes del primer pago: el que diga la persona ("el primer pago es en diciembre"); si no, el mes siguiente a la compra.
  const primer = /^\d{4}-\d{2}$/.test(String(primerPago || '')) ? String(primerPago) : mesSiguiente_(new Date());
  return { producto: String(producto || 'Compra en cuotas').trim(), total: total, n: n, cuota: cuota, primer: primer };
}
```
