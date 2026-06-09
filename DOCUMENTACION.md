# Documentación — Invitación Boda Carlos & Jimena

> Última actualización: 8 de junio de 2026
> Repositorio: `Anibru300/Invitacion-a-mi-Boda` (GitHub Pages)
> URL pública: `https://anibru300.github.io/Invitacion-a-mi-Boda/`

---

## 1. Estructura del proyecto

Es un sitio **estático de un solo archivo** (`index.html`). No hay build step ni framework.

```
Invitacion-a-mi-Boda/
├── index.html          ← Todo el sitio aquí
├── images/
│   ├── principal.jpeg
│   ├── portada.jpeg
│   ├── Carlos y Jimena.jpeg
│   ├── momento1-16.jpeg
│   └── ...
└── DOCUMENTACION.md    ← Este archivo
```

### Stack
- HTML5 + Tailwind CSS (CDN)
- JavaScript vanilla (todo en `<script>` al final del body)
- Lucide icons (CDN)
- canvas-confetti (CDN)
- html2canvas (CDN)
- YouTube IFrame API (reproductor de música)

---

## 2. Flujo general del invitado

1. El invitado abre el link personalizado o el link general.
2. Ve la **portada (hero)** con el botón "Abrir Invitación".
3. Al abrir, aparece el contenido: countdown, padres, padrinos, itinerario, galería, RSVP, etc.
4. En la sección **RSVP** confirma asistencia.
5. Si confirma **Sí**, aparece el **Pase de Acceso** con el número de personas.
6. Puede descargar el pase como imagen PNG.

---

## 3. Sistema de pases pre-asignados (control de invitados)

### Cómo se genera un link personalizado

Entra al **panel de administración**:
```
https://anibru300.github.io/Invitacion-a-mi-Boda/?admin=1
```

Aparece al final de la página un apartado **"Generador de Invitaciones"**.

1. Escribe el nombre del invitado.
2. Selecciona el número de pases (1–5, o más si se modifica el HTML).
3. Click en **"Generar link personalizado"**.
4. Copia el link y envíalo por WhatsApp.

El link generado se ve así:
```
https://anibru300.github.io/Invitacion-a-mi-Boda/?invitado=Abuelita%20Mar%C3%ADa&pases=6
```

### Cómo lo ve el invitado

- Su nombre aparece **pre-lleno y bloqueado** en el campo de nombre.
- En el RSVP, al seleccionar **"Sí, asistiré"**, aparecen botones del **1 al máximo asignado**.
- Puede elegir cuántas personas realmente asistirán (no puede elegir más del máximo).
- Por defecto se pre-selecciona el máximo asignado.
- El **Pase de Acceso** mostrará exactamente el número que eligió.

### Lógica clave en el código

```javascript
// Línea ~1693
const urlParams = new URLSearchParams(window.location.search);
const preAssignedName = urlParams.get('invitado') || '';
const preAssignedPax  = urlParams.get('pases') || '';
```

Si hay `preAssignedPax`, se generan botones dinámicos (1 a N) en el contenedor `#pax-buttons`.

---

## 4. RSVP y Google Forms

### Campos que se envían a Google Forms

| Campo | entry ID | Descripción |
|-------|----------|-------------|
| Nombre | `entry.837092975` | Nombre del invitado |
| Asistencia | `entry.764767312` | "Si" o "No" |
| Pax | `entry.706411984` | Número de personas |
| Notas | `entry.1995459750` | Mensaje opcional |

### Cómo se envía

Se usa `navigator.sendBeacon()` con fallback a `fetch(no-cors)`. El envío es "fire and forget" — no esperamos respuesta del servidor de Google.

```javascript
const BASE = 'https://docs.google.com/forms/d/e/1FAIpQLSf1gQvRgM63JLVnElM8zpV4Qm2TBFll-OOeuWBIBJauKLa-Rw/formResponse';
```

> **Importante:** No cambiar las entry IDs a menos que se reconecte un formulario nuevo.

---

## 5. Bloqueo de una sola confirmación (localStorage)

### Problema que resuelve

Evitar que el mismo invitado confirme dos veces o genere múltiples pases desde el mismo dispositivo/navegador.

### Cómo funciona

1. Al enviar el RSVP exitosamente, se guarda en `localStorage`:
   ```javascript
   localStorage.setItem('boda_cj_<hash>', JSON.stringify({
     attendance: 'Si',   // o 'No'
     pax: '4',
     name: 'Abuelita María',
     timestamp: 1717891200000
   }));
   ```
   La clave (`storageKey`) se genera a partir del nombre del invitado.

2. Si el invitado **recarga la página**, el script detecta la clave guardada y:
   - Auto-abre la invitación (salta el hero).
   - Oculta el formulario RSVP.
   - Muestra el mensaje de éxito correspondiente.
   - Si confirmó **"Sí"**, muestra el pase de acceso automáticamente.

3. **No puede volver a enviar el formulario** ni generar otro pase desde ese navegador.

### Para "resetear" una confirmación (pruebas)

Abrir la consola del navegador (F12 → Console) y ejecutar:
```javascript
// Listar todas las claves guardadas
Object.keys(localStorage).filter(k => k.startsWith('boda_cj_'))

// Borrar una específica (copia la clave del paso anterior)
localStorage.removeItem('boda_cj_XXXXX')

// Borrar TODAS las confirmaciones
Object.keys(localStorage).forEach(k => { if(k.startsWith('boda_cj_')) localStorage.removeItem(k); });
```

---

## 6. Pase de Acceso (Ticket)

### Cuándo aparece

- **Primera vez:** Después de confirmar "Sí", automáticamente tras ~800ms de confeti.
- **Recargas posteriores:** Aparece inmediatamente si hay confirmación guardada en `localStorage`.

### Qué muestra

- Nombres de los novios
- Fecha
- **Número exacto de personas** que el invitado confirmó
- Disclaimer de validez

### Descarga

El botón "Descargar pase" usa `html2canvas` para convertir el elemento `#pase-acceso` en una imagen PNG:
```javascript
html2canvas(ticketElement, { backgroundColor: '#060913', scale: 3, useCORS: true })
```

---

## 7. Panel de Admin (`?admin=1`)

### Cómo funciona

1. Se detecta el parámetro `?admin=1` al cargar.
2. El loading screen se oculta más rápido (~300ms).
3. Se ejecuta `openInvitation(true)` automáticamente (salta el hero, sin música).
4. Se hace visible la sección `#admin-panel` y se hace scroll hacia ella.

### Funciones del admin

```javascript
setAdminPax(value)        // Resalta el botón de pases seleccionado
generateInviteLink()      // Construye el link con invitado + pases
copyAdminLink()           // Copia al portapapeles
```

### Para agregar más botones de pases en el admin

Buscar en el HTML (~línea 1088):
```html
<button type="button" onclick="setAdminPax('6')" class="admin-pax-btn ...">6</button>
```
Agregar botones similares dentro del `grid`.

---

## 8. Secciones del sitio (en orden)

1. **Loading screen** — spinner dorado
2. **Hero / Portada** — imagen de fondo + sello + "Abrir Invitación"
3. **Navbar** — aparece al hacer scroll
4. **Inicio** — mensaje de bienvenida
5. **Padres y Padrinos**
6. **Lugar y Fecha** — cards de iglesia y salón
7. **Itinerario** — timeline vertical
8. **Código de Vestimenta**
9. **Lluvia de Sobres**
10. **Galería** — grid de fotos + lightbox
11. **RSVP** — formulario de confirmación
12. **Pase de Acceso** — ticket (aparece después de confirmar)
13. **Compartir** — QR + botones copiar/compartir
14. **Reproductor de música** — YouTube (esquina inferior)

---

## 9. Cómo hacer cambios y desplegar

### Editar

Todo está en `index.html`. Abrir el archivo, buscar la sección relevante, modificar.

### Commit y push

```bash
cd Invitacion-a-mi-Boda
git add index.html
git commit -m "describir cambio"
git push origin main
```

GitHub Pages despliega automáticamente desde la rama `main`. Tarda 1–2 minutos en reflejarse.

### Probar localmente

Simplemente abrir `index.html` en el navegador:
```bash
cd Invitacion-a-mi-Boda
# En Windows:
start index.html
# O arrastrar el archivo a una pestaña del navegador
```

Para probar parámetros de URL localmente, abrir:
```
file:///C:/.../Invitacion-a-mi-Boda/index.html?admin=1
```

---

## 10. Cosas a tener en cuenta para el futuro

- **Google Forms:** Si se necesita cambiar el formulario, hay que actualizar la URL `BASE` y las `entry.XXXXXX` IDs. Se obtienen inspeccionando el HTML del formulario de Google.
- **Imágenes:** Si se cambian fotos, reemplazar los archivos en `/images/` manteniendo los mismos nombres, o actualizar las rutas en `index.html`.
- **localStorage:** Si un invitado cambia de navegador/dispositivo o borra datos, podría confirmar de nuevo. El "bloqueo" es a nivel de dispositivo, no de servidor.
- **Máximo de pases:** El admin genera links con `pases=N`. El RSVP genera botones 1 a N. Si N > 5, los botones se acomodan en múltiples filas por el grid de 5 columnas.
- **Música:** El reproductor usa la API de YouTube IFrame. Si el video deja de existir, hay que actualizar el `videoId` en la función `initMusicPlayer()`.

---

## 11. Contacto / Quién hizo qué

- Desarrollo y modificaciones: Kimi Code CLI
- Pareja: Carlos & Jimena
- Fecha del evento: 29 de agosto de 2026
