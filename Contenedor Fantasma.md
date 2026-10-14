# Contenedor Fantasma

## 1. Datos del reto

| Campo | Valor |
|-------|-------|
| **Nombre** | Contenedor Fantasma |
| **Categoría** | Criptografía |
| **Duración** | Ventana flash de 50 minutos |
| **Objetivo** | Descifrar el contenedor y extraer la llave de conciliación |
| **Formato de entrega** | `T4C{...}` |
| **Flag** | `T4C{m3rs3nn3_tw1st3r_s33d_d3b1l}` |

---

## 2. Contexto del reto

> "En el servidor aparece un contenedor cifrado con manifiestos sensibles. Lo generó el mismo appliance del proveedor, que en lugar del generador criptográfico del sistema usa el del propio lenguaje y lo reinicializa en cada arranque a partir del reloj.
>
> El contenedor publica en su cabecera un valor que salió de ese mismo material.
>
> Las marcas de tiempo del sistema de ficheros sitúan su creación dentro de una ventana de cuatro horas.
>
> Misión: descifrar el contenedor y extraer la llave de conciliación.
>
> La ventana no da para probar a ciegas, pero sí para una búsqueda bien dirigida.
>
> Formato de entrega: `T4C{...}`. Es la llave de conciliación, que está en uno de los ficheros de texto del contenedor descifrado."

---

## 3. Cadena del exploit

El contenedor estaba cifrado con **AES-CBC**, pero la clave y el nonce de la cabecera no salían del generador criptográfico del sistema: salían del **PRNG por defecto del lenguaje** (Mersenne Twister), **re-sembrado en cada arranque con la hora del reloj del sistema**.

### 3.1. Reducción del espacio de búsqueda

La ventana de 4 horas (2026-02-13 22:00 UTC → 2026-02-14 02:00 UTC) solo da **14 400 segundos posibles de semilla** — totalmente abordable en segundos, nada de fuerza bruta "a ciegas".

### 3.2. Ancla de verificación

Se probó cada semilla candidata re-sembrando el PRNG y generando el primer valor de 64 bits. Solo una semilla (**1771029663**) reproducía exactamente el nonce publicado en la cabecera (**0x1db7722afc70aa7f**) — estadísticamente imposible por azar (1 entre 2⁶⁴).

### 3.3. Reconstrucción de la clave

Con esa misma semilla, el siguiente material del PRNG (256 bits, extraídos byte a byte) daba la **clave AES-256 correcta**.

### 3.4. Descifrado del cuerpo

Se descifró el cuerpo con esa clave y el IV publicado en la cabecera (ese sí era aleatorio genuino) → se obtuvo un archivo **ZIP válido** (`PK\x03\x04`).

### 3.5. Contenido del ZIP

Dentro del ZIP:

- `manifiesto_convoy.txt`
- `frecuencias_enlace.txt`
- `llave_conciliacion.txt` (el objetivo)

---

## 4. Llave de conciliación

    T4C{m3rs3nn3_tw1st3r_s33d_d3b1l}

La propia clave confirma la vulnerabilidad: una semilla débil de Mersenne Twister derivada del reloj.

---

## 5. Lecciones aprendidas

- **No uses el PRNG del lenguaje para criptografía.** Mersenne Twister no es criptográficamente seguro.
- **Una semilla derivada del reloj es predecible.** Con una ventana de tiempo conocida, el espacio de búsqueda se reduce drásticamente.
- **El nonce publicado en la cabecera sirve como ancla de verificación.** Si una semilla reproduce ese valor, es la correcta.
- **El cifrado híbrido (AES + RSA) es común**, pero aquí el fallo estaba en la generación de la clave simétrica.
- **La ventana de tiempo es una pista clave.** No hay que probar a ciegas, sino dirigir la búsqueda.
