# Gobernanza IA — Extensión de navegador (DLP de IA)

Extensión **Manifest V3** (Chrome / Edge / Brave) del producto Cognitive Governance. Protege los datos de la
empresa en el canal donde ocurre la mayoría de las fugas: **lo que un empleado escribe en una herramienta de IA
web** (ChatGPT, Claude, Gemini, DeepSeek, etc.).

No es un bloqueador: deja trabajar, pero **enmascara los datos sensibles antes de que salgan del navegador**,
avisa, puede cancelar la descarga de instaladores de apps de IA, y reporta al central **solo eventos ya
enmascarados** (nunca el dato real).

---

## Qué hace

1. **Enmascara datos sensibles** en el mensaje antes de enviarlo a la IA: correo, teléfono, RFC, CURP, CLABE,
   tarjeta (validadas con su dígito verificador) y los **términos confidenciales de la empresa** → se reemplazan
   por `[EMAIL]`, `[TARJETA]`, `[CONFIDENCIAL]`, etc.
2. **Avisa** cuando detecta datos sensibles y deja al usuario decidir (enviar / ocultar).
3. **Cancela/pausa descargas** de instaladores de apps de IA de escritorio (ChatGPT, Claude, Ollama, LM Studio…).
4. **Reporta al central** (Lab 2) los eventos con datos sensibles, ya enmascarados, y el inventario de uso web de
   IA (sitios del catálogo, visitas, minutos).

---

## Cómo funciona (flujo)

```
  Usuario escribe en chatgpt.com
          │
          ▼
  content.js detecta el envío → redact.js enmascara el texto
          │ (si hay datos sensibles)  ──►  warning.html: avisar / ocultar / enviar
          ▼
  background.js  ──► outbox en chrome.storage.local
          │ (si hay serverUrl+token+hostname)
          ▼
  POST /api/agent/prompt-events  (Bearer token)  ──►  Central (consola de Seguridad)
```

**Sin configuración de servidor**, la extensión funciona como **recolector local**: enmascara y avisa, pero no
envía nada. En cuanto recibe `serverUrl + token + hostname`, empieza a reportar.

---

## Archivos

| Archivo | Rol |
| --- | --- |
| `manifest.json` | Manifiesto MV3: permisos, service worker, popup, esquema de política. |
| `background.js` | Service worker: outbox y envío al central, guardia de descargas, detector de sitios de IA. |
| `content.js` | Se inyecta en la página: captura el envío del mensaje y lo pasa a enmascarar. |
| `redact.js` | Enmascarado DLP (regex + validaciones Luhn/CLABE) y términos de la empresa. |
| `detector.js` | Señales para reconocer que una pestaña es una herramienta de IA. |
| `ai_domains.js` | Catálogo fijo de dominios de IA conocidos. |
| `catalog.js` | Apoyo de catálogo para el popup. |
| `warning.html/.css/.js` | Aviso en pantalla cuando hay datos sensibles. |
| `popup.html/.css/.js` | Ventana de la extensión: Inventario, Por día, Prompts, Términos y **Conexión**. |
| `schema.json` | Esquema de la política administrada (`chrome.storage.managed`). |
| `install-policy.ps1` | Configura la extensión por GPO/Intune en toda una flota (zero-touch). |

---

## Permisos (y por qué)

| Permiso | Para qué |
| --- | --- |
| `storage` | Config (local/managed) y buffer de eventos (outbox). |
| `downloads` | Pausar/cancelar descargas de instaladores de IA. |
| `tabs`, `scripting` | Saber en qué sitio está el usuario e inyectar la lógica de captura. |
| `webRequest`, `declarativeNetRequest` | Apoyo a la detección de uso de IA. |
| `alarms` | Reintentar el envío del outbox periódicamente. |
| `host_permissions: <all_urls>` | La IA puede estar en cualquier dominio; el enmascarado debe correr donde sea. |

---

## Configuración

La extensión lee su config en este orden de prioridad: **política administrada** (`chrome.storage.managed`) →
**almacenamiento local** (popup, para pruebas) → vacío (recolector local).

Claves (ver `schema.json`):

| Clave | Obligatoria | Qué es |
| --- | --- | --- |
| `serverUrl` | sí (para reportar) | Base de la consola, ej. `https://consola.midominio.com`. |
| `token` | sí (para reportar) | Token **por organización** (el mismo del agente de Windows, `lab2_...`). |
| `hostname` | sí (para reportar) | Nombre del equipo; debe coincidir con el del agente para unir ambos reportes. |
| `user` | opcional | `DOMINIO\usuario`. Si falta, se conserva la persona que asignó el agente. |
| `area` | opcional | Área del equipo. |
| `orgTerms` | opcional | Lista de términos confidenciales (clientes, proyectos, códigos) → `[CONFIDENCIAL]`. |

---

## Instalación

### A) Zero-touch, por política (flotas / TI) — recomendado
Desde Intune, GPO o el instalador de TI (como administrador):
```powershell
powershell -ExecutionPolicy Bypass -File install-policy.ps1 `
  -ExtensionId <id-de-32-letras> -ServerUrl https://consola.midominio.com -Token lab2_xxx -ForceInstall
```
Empuja `serverUrl + token + hostname (=%COMPUTERNAME%)` por política y, con `-ForceInstall`, instala la extensión.
Quitar: `install-policy.ps1 -ExtensionId ... -Remove`.

### B) Manual / pruebas (equipos sueltos)
1. `chrome://extensions` → activa **Modo desarrollador** → **Cargar descomprimida** → elige esta carpeta.
2. Abre el popup → pestaña **Conexión** → pega `serverUrl`, `token` y `hostname`, prueba la conexión y guarda.

> La prueba de conexión usa `GET /api/agent/web-domains` con el token: sirve para confirmar que el token es válido.

---

## Contrato con el central (Lab 2)

- **`POST /api/agent/prompt-events`** — eventos DLP. Autenticado con `Authorization: Bearer <token>`.
  - El `text` va **siempre enmascarado**; el dato real nunca sale del navegador.
  - **Minimización:** solo se envía/guarda si el mensaje traía datos sensibles o pasó por el aviso.
  - **Idempotencia:** cada evento lleva un `id`; el central ignora duplicados.
  - **Outbox:** el evento se guarda local, se envía y solo se borra al confirmarse (200); si falla, reintenta.
- **`GET /api/agent/web-domains`** — dominios de IA que vigila (y prueba de token).

---

## DLP: qué enmascara

- **Patrones:** `EMAIL`, `TELÉFONO`, `RFC`, `CURP`, `CLABE`, `TARJETA`. Los números (tarjeta/CLABE) se validan
  con su dígito verificador para no ocultar cualquier cifra.
- **Términos de la empresa (`orgTerms`):** lo que coincida se reemplaza por `[CONFIDENCIAL]`.
- **Guardia de descargas:** un instalador (`.exe`, `.msi`, `.dmg`, `.pkg`…) cuyo nombre/URL coincide con una app
  de IA conocida se **pausa** y el usuario decide cancelar o continuar.
- **Detector de sitios de IA:** suma señales de la pestaña; con umbral `DETECT_THRESHOLD = 2` basta una caja de
  chat de IA clara para marcar el sitio.

---

## Alcance y límites

- Cubre el **canal navegador**. Las **apps de escritorio** de IA (nativas) no las ve una extensión: ese canal se
  cierra con controles del cliente (bloqueo por AppLocker/Intune, DNS para inventario), no con este código.
- El enmascarado depende de patrones: texto libre no estructurado puede escaparse (límite conocido y asumido).
- No publica en la tienda ni hace proxy/CASB de tráfico (fuera de alcance por ahora).

---

*Parte del producto Cognitive Governance. La central (consola + API) vive en el repositorio principal; esta
extensión es uno de sus colectores de entrada.*
