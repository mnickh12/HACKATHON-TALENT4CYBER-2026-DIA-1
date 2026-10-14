cat > "Manifiesto Cifrado.md" << 'EOF'
# Manifiesto Cifrado — PIR-01

## 1. Contexto y pista falsa

La nota del analista sugería dos caminos "oficiales":

1. Buscar la clave en bases de claves débiles conocidas.
2. Lanzar una factorización directa de fuerza bruta sobre un módulo de 2048 bits.

El propio enunciado avisaba de que esa anotación **no estaba verificada** y había que tratarla como cualquier fuente sin contrastar.

Factorizar un RSA-2048 por fuerza bruta es **computacionalmente inviable** (tardaría más que la edad del universo), así que esa pista era un **señuelo**.

La pista real estaba en otro sitio: el proveedor genera las claves en un **appliance de bajo coste**, y las dos claves (A y B) se emitieron con **pocos minutos de diferencia**. Esto es la "bandera roja" clásica de una vulnerabilidad conocida en generación de claves RSA con **baja entropía**.

---

## 2. La vulnerabilidad: primos compartidos entre claves RSA

Una clave RSA se construye eligiendo dos primos grandes `p` y `q` al azar, y el módulo público es `n = p × q`.

Si el generador de números aleatorios del dispositivo tiene **poca entropía** (algo común en hardware barato, sobre todo justo al arrancar, cuando el sistema aún no ha acumulado "ruido" suficiente), puede ocurrir que dos ejecuciones cercanas en el tiempo **reutilicen uno de los dos primos**. Es decir:

    n_A = p × q_A
    n_B = p × q_B      (mismo p, distinto q)

Esto es **catastrófico** para la seguridad, porque calcular el **máximo común divisor (GCD)** de dos números es trivial computacionalmente (algoritmo de Euclides, milisegundos) aunque los números tengan miles de dígitos. Si dos módulos comparten un factor, `gcd(n_A, n_B)` devuelve directamente ese primo compartido.

Esto contrasta con factorizar un **único** `n` de 2048 bits, que es inviable: la dificultad de RSA reside en que no hay ningún atajo conocido para factorizar un número grande si no tienes información adicional. Pero aquí **sí la había**: el segundo módulo.

---

## 3. Pasos técnicos

### a) Extraer los módulos de las dos claves públicas

Cargué ambos `.pem` con la librería `cryptography` de Python y extraje `n` y `e` de cada una (las dos usaban exponente estándar `e = 65537`).

### b) Calcular el GCD

    import math
    g = math.gcd(n1, n2)

El resultado no fue `1`, sino un número de **1024 bits exactamente** — confirmando que ese es el primo `p` compartido entre ambas claves.

### c) Reconstruir el segundo primo

Con `p` conocido, el otro factor sale por simple división:

    q1 = n1 // p

Comprobé que `p × q1 == n1` ✅ (y lo mismo con `n2`), confirmando que la factorización es correcta.

### d) Reconstruir la clave privada completa

Teniendo `p`, `q`, `e` y `n`, se puede calcular todo lo necesario para una clave privada RSA completa:

- `φ(n) = (p-1)(q-1)` (función de Euler)
- `d = e⁻¹ mod φ(n)` (inverso modular del exponente público → exponente privado)
- `dmp1 = d mod (p-1)`, `dmq1 = d mod (q-1)`, `iqmp = q⁻¹ mod p` (parámetros CRT para acelerar el descifrado, formato estándar)

Con esto monté un objeto `RSAPrivateNumbers` operativo, equivalente a tener el `.pem` privado original.

### e) Descifrar el sobre híbrido

El formato del `.enc`, según la nota del analista, era:

    [ 256 bytes ]  clave de sesión AES envuelta con RSA-OAEP
    [  16 bytes ]  IV
    [   resto   ]  cuerpo cifrado en AES-256-CBC (padding PKCS#7)

Este es un esquema de **cifrado híbrido**: RSA es lento para cifrar datos grandes, así que solo se usa para proteger una clave simétrica corta (aquí, una clave AES-256), y el contenido real se cifra con AES, que es mucho más rápido.

Con la clave privada reconstruida, descifré los primeros 256 bytes usando **RSA-OAEP**. Aquí tuve un pequeño obstáculo: OAEP necesita especificar qué función hash usa internamente (para el MGF1 y el propio padding). Probé primero con **SHA-256** (el estándar más moderno) y falló; al probar con **SHA-1** (más antiguo, típico de implementaciones "de bajo coste" o legacy) funcionó correctamente y obtuve una clave de **32 bytes** (justo el tamaño de AES-256).

### f) Descifrar el cuerpo AES

Con la clave de sesión y el IV, descifré el resto en modo **CBC** y quité el padding **PKCS#7** (que simplemente rellena el último bloque indicando en el propio valor del byte de relleno cuántos bytes hay que quitar).

---

## 4. Resultado

El texto en claro era el **manifiesto de carga real**, donde se revelaba que el contenedor **MLIU 774139-2** tenía una referencia de envío pública falsa, y la referencia verdadera (el flag) era:

    T4C{f4ct0r_c0mun_ENV_4LG_77413}

---

## 5. Lecciones aprendidas

- **No toda la información del enunciado es fiable.** La pista de "factorización por fuerza bruta" era un señuelo.
- **La baja entropía en la generación de claves RSA es un problema real** (appliances baratos, arranques en frío, etc.).
- **El GCD es una herramienta potentísima** cuando dos módulos RSA comparten un primo.
- **RSA-OAEP con SHA-1** sigue apareciendo en implementaciones legacy.
- **El cifrado híbrido** (RSA + AES) es el estándar para proteger datos grandes.
EOF
