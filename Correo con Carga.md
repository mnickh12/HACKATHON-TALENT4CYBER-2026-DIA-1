# Correo con Carga — PIR-04

## 1. Datos del reto

| Campo | Valor |
|-------|-------|
| **Nombre** | PIR-04. Vector de entrada |
| **Categoría** | Forense |
| **Objetivo** | Analizar el correo y el adjunto para extraer la configuración del segundo estadio |
| **Formato de entrega** | `T4C{...}` |
| **Flag** | `T4C{m4cr0_st4g3r_b4s364_0culto}` |

---

## 2. Resumen

El reto tiene dos partes y cada una se resuelve con una técnica distinta:

1. **Cabeceras del `.eml`**: demostrar que el remitente es falso.
2. **Adjunto `.docm`**: sacar la macro VBA, desofuscar la carga y leer el campo `marcador`.

Todo el análisis fue **estático**. No se ejecutó la macro ni se abrió el documento en Office.

---

## 3. Análisis del correo

### 3.1. El remitente está falsificado

El correo se presenta como `Sistemas MLI <sistemas@mli-iberia.example>`, pero las cabeceras lo contradicen:

| Cabecera | Qué muestra |
|----------|-------------|
| **Received más antigua** | Sale de `PC-OPERACIONES` hacia `mail-out-3.transportes-delta.example` (IP `185.203.117.44`). Después entra en `mx01.mli-iberia.example`. Nunca pasó por la infraestructura de MLI. |
| **Return-Path** | `bounce+mli@transportes-delta.example`, un dominio ajeno al de `From`. |
| **Authentication-Results del MX receptor** | `spf=fail`, `dkim=fail`, `dmarc=fail` (quarantine). |
| **Otra Authentication-Results con todo en pass** | Falsa: está colocada entre los `Received` del atacante, así que la inyectó el emisor. La válida es la que añade el MX receptor (`mx01.mli-iberia.example`), y esa dice `fail`. |
| **DKIM-Signature** | Declara `d=mli-iberia.example`, pero el servidor receptor la verificó y no pasa. Es un intento de aparentar firma legítima. |
| **Message-ID** | Usa `@mail.mli-iberia.example`, dominio que no coincide con el servidor emisor real. |
| **Ingeniería social** | Pide abrir el adjunto y "habilitar el contenido" antes del cierre de turno, con la etiqueta `[ACCION REQUERIDA]`. |

**Conclusión:** el `From` dice `sistemas@mli-iberia.example`, pero el correo **nunca pasó por la infraestructura de MLI**. Se envió desde un servidor de `transportes-delta.example`. Eso acredita la suplantación.

> **Regla práctica:** la única `Authentication-Results` fiable es la que añade tu servidor receptor. Cualquier otra que venga "de fábrica" en el correo puede ser forjada.

### 3.2. Cómo se leen las cabeceras Received

Se leen **de abajo arriba**, porque cada servidor añade la suya por encima. La más antigua es la de abajo y muestra de dónde salió realmente el correo.

---

## 4. Análisis del adjunto `.docm`

Un `.docm` es un ZIP con XML dentro (formato OOXML). Al descomprimirlo, se encuentra:

    word/document.xml        ← texto visible (el señuelo)
    word/vbaProject.bin      ← aquí está la macro
    word/_rels/document.xml.rels
    [Content_Types].xml

- `document.xml` contiene el señuelo: pide "habilitar el contenido activo". Es ingeniería social para que el usuario permita la macro.
- `vbaProject.bin` es un contenedor OLE con el código VBA comprimido. Lo que aparece al hacer `strings` solo son trozos sueltos.

### 4.1. Extracción del VBA

Normalmente se usaría `olevba` (de `oletools`), pero no estaba instalado y el sandbox no tiene red. Así que se descomprimió el código **a mano**.

**Cómo funciona la compresión (MS-OVBA):**

- El flujo del módulo empieza con un byte `0x01` (firma).
- Después van *chunks*. Cada uno tiene una cabecera de 2 bytes que indica su tamaño.
- Dentro de cada *chunk* hay bloques de 8 elementos precedidos por un byte de flags. Un bit a `0` significa "byte literal" y un bit a `1` significa "token de copia" (longitud y desplazamiento hacia atrás en lo ya descomprimido).

**Cómo se localizó el inicio:** se buscó la cadena `Attribut` (de `Attribute VB_Name`) y se retrocedió hasta el `0x01`, que estaba en el offset 896. Desde ahí se aplicó el algoritmo de descompresión.

El resultado fue el código fuente completo de `Module1`.

### 4.2. Entender la macro

    Private Const DESTINO = "svc_upd.dat"

    Ensamblar()    ' concatena 6 fragmentos y aplica StrReverse
    Decodificar()  ' base64 → bytes, usando MSXML2.DOMDocument
    Desplegar()    ' escribe el resultado en %TEMP%\svc_upd.dat

    Sub AutoOpen()        : Desplegar
    Sub Document_Open()   : Desplegar

Es un **dropper**. Al abrir el documento (`AutoOpen` / `Document_Open`) reconstruye una carga ofuscada y la deja en disco.

La ofuscación son dos capas:

1. Cadena partida en trozos, para que ningún string completo salga en una búsqueda simple.
2. Cadena invertida y luego base64.

Que la cadena esté invertida se nota porque **empieza por `==`**: el relleno de base64 va al final, así que si aparece al principio, está al revés.

### 4.3. Desofuscar sin ejecutar nada

Se reprodujo en Python exactamente lo que haría la macro, pero solo con transformaciones de texto:

    import base64
    partes = [...]                      # los 6 fragmentos copiados de Ensamblar()
    s = "".join(partes)[::-1]           # equivale a StrReverse
    print(base64.b64decode(s + "="*(-len(s) % 4)).decode("latin1"))

**Salida (la configuración del segundo estadio):**

    canal      = cdn.transportes-delta.example
    intervalo  = 900
    identidad  = MLI-ALG-0774
    mutex      = Global\NavalSyncUpdater
    marcador   = T4C{m4cr0_st4g3r_b4s364_0culto}

El flag es el campo `marcador`: `T4C{m4cr0_st4g3r_b4s364_0culto}`.

---

## 5. Cadena de infección

    Correo con remitente falsificado (transportes-delta.example)
       → adjunto .docm con señuelo "habilita el contenido"
          → macro AutoOpen / Document_Open
             → ensambla + invierte + decodifica base64
                → escribe %TEMP%\svc_upd.dat (config del 2º estadio)

---

## 6. IOCs útiles para el informe

- **IP:** `185.203.117.44`
- **Dominio:** `transportes-delta.example`
- **Fichero:** `%TEMP%\svc_upd.dat`
- **Mutex:** `Global\NavalSyncUpdater`
- **Referencia interna del documento:** `MLI-SIS-2026-0119`

---

## 7. Lecciones aprendidas

- **Las cabeceras `Received` se leen de abajo arriba.** La más antigua es la que muestra el origen real.
- **La única `Authentication-Results` fiable es la que añade tu servidor receptor.** Las que vienen "de fábrica" pueden ser forjadas.
- **Un `.docm` es un ZIP.** Se puede analizar sin abrirlo en Office.
- **El VBA se puede descomprimir a mano** (MS-OVBA) si no hay herramientas disponibles.
- **La ofuscación por inversión + base64** se detecta porque el relleno `=` aparece al principio.
- **El análisis estático es suficiente** para extraer la configuración sin ejecutar la macro.
