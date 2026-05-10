# Patrón: Loop Habitante con Atribución

**Origen:** Frente `p10_t10` · ciclos 12–15 (10-may-2026)
**Estado:** Validado en producción (P10 + T10 · commits b54e31a → eb8259f)
**Aplicabilidad:** Cualquier landing con formulario de captura de leads (`liveodi`, `boton_turismo`, futuras verticales)

## Qué resuelve

El habitante recién inscrito en una landing queda en estado **pasivo** después del submit:
"Te avisaremos…" + nada que hacer. El loop habitante→nuevo-habitante no existe o no es atribuible.

Este patrón:
1. Activa una acción inmediata de virality post-submit (Web Share API + clipboard fallback)
2. Inyecta atribución UTM al URL compartido para que el receptor que se inscriba quede marcado
3. Cierra el loop sin tocar backend ni schema BD (puramente frontend)

## Componentes

### 1. Botón Compartir en el success state

```html
<div id="lead-success" class="form-success" style="display:none">
  <h3>✓ ¡Listo! ¡Estás en la lista!</h3>
  <p>[copy específico de la vertical]</p>
  <p data-i18n="success_share_intro" style="margin-top:18px;font-size:0.9rem;color:#444;">
    ¿Conoces a alguien que también [acción específica]?
  </p>
  <button type="button" id="lead-share" data-i18n="success_share_btn"
    style="margin-top:8px;padding:10px 22px;border:0;border-radius:24px;
    background:var(--brand-primary);color:#fff;font-weight:600;cursor:pointer;">
    Compartir con un [colega/amigo/contacto]
  </button>
  <p id="lead-share-feedback" style="margin-top:10px;font-size:0.85rem;display:none;"></p>
</div>
```

### 2. Lógica JS con Web Share API + clipboard fallback + UTM atribución

```js
document.getElementById('lead-share').addEventListener('click', async () => {
  const url = window.location.origin + window.location.pathname
            + '?utm_source=habitant_share&utm_medium=referral';
  const title = document.title;
  const text = T('success_share_intro');
  const fb = document.getElementById('lead-share-feedback');
  try {
    if (navigator.share) {
      await navigator.share({ title, text, url });
      fb.textContent = T('success_share_thanks');
      fb.style.display = 'block';
    } else if (navigator.clipboard) {
      await navigator.clipboard.writeText(url);
      fb.textContent = T('success_share_copied');
      fb.style.display = 'block';
      setTimeout(() => { fb.style.display = 'none'; }, 4000);
    }
  } catch (e) { /* user cancelled share — silent */ }
});
```

### 3. Form submit ya envía utm_source/utm_medium al backend

El form de captura debe leer `URLSearchParams` y enviar al backend:

```js
const params = new URLSearchParams(window.location.search);
fetch(API + '/api/leads/capture', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    name, email, phone, city, /* ... */
    source: "landing", source_url: window.location.href,
    utm_source: params.get('utm_source'),    // ← CRÍTICO: capturar
    utm_medium: params.get('utm_medium'),    // ← CRÍTICO: capturar
    utm_campaign: params.get('utm_campaign'),
    referral_code: params.get('ref')
  })
});
```

## Resultado runtime

- Habitante A se inscribe → ve botón "Compartir"
- A click share → URL al portapapeles/WhatsApp/Twitter con `?utm_source=habitant_share&utm_medium=referral`
- Receptor B llega al landing → se inscribe → backend recibe lead con `utm_source=habitant_share`
- Arquitecto consulta BD: `SELECT count(*) FROM odi_leads WHERE utm_source='habitant_share'` → métrica de virality real
- Futuro: reconocimiento de referidos atribuibles

## Verificación E2E

```js
// Playwright headless: clipboard.readText() después de click
const result = await page.evaluate(async (id) => {
  document.getElementById(id+'-form').style.display = 'none';
  document.getElementById(id+'-success').style.display = 'block';
  document.getElementById(id+'-share').click();
  await new Promise(r => setTimeout(r, 500));
  return await navigator.clipboard.readText();
}, formId);
// Esperado: "https://<dominio>/?utm_source=habitant_share&utm_medium=referral"
```

## Adopción esperada

- `liveodi` — landing liveodi.com si captura leads
- `boton_turismo` — landings de turismo dental con formulario paciente
- `odi_humans` — perfiles habitante, sección "invita a otro"
- Futuras verticales con landing + formulario de captura

## Doctrinas aplicadas

- **#ND-11 ENTREGA candidata**: activa modo Habitante con acción inmediata de valor real
- **#ND-13 HABLA candidata**: opera en modo HABITANTE (post-submit) sin confundir con ANÓNIMO/ARQUITECTO
- **#ND-12 RESPIRA FIRME**: trabajo entregado async sin bloquear ciclo

## Fuente

- P10 prod: `https://www.profesionales10.online/`
- T10 prod: `https://www.teveo10.club/`
- Commits: `a8147de` (botón) · `5fdc324` (UTM atribución)
- Repo: `odi-landings` rama `main`
