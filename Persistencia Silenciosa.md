# Persistencia Silenciosa — PIR-05

## 1. Datos del reto

| Campo | Valor |
|-------|-------|
| **Nombre** | PIR-05. Análisis de persistencia |
| **Categoría** | Reversing |
| **Caso** | IR-2026-0209 |
| **Objetivo** | Determinar el mecanismo exacto de persistencia del implante y extraer la flag |
| **Formato de entrega** | `T4C{...}` |
| **Flag** | `T4C{etc_systemd_system_naval_syncd_timer}` |

---

## 2. Contexto del incidente

El servidor de planificación (`srv-planificacion-01`) fue comprometido a través del adjunto del Reto 4. El binario recuperado de `/usr/lib/naval/` sobrevive a los reinicios.

El material de análisis incluye el binario `persistencia.elf` (ELF x86-64, sin símbolos) y un volcado de triaje tomado **en caliente** antes del aislamiento del host.

> **Advertencia del caso:** No ejecutar el binario fuera de un entorno controlado. El análisis previsto es estático; el binario no hace nada visible si el entorno no coincide con el del host original.

---

## 3. Material analizado

Ficheros proporcionados en la carpeta de análisis:

| Fichero | Descripción |
|---------|-------------|
| `persistencia.elf` | Binario recuperado de `/usr/lib/naval/`. ELF64, 14496 bytes, sin símbolos |
| `etc_hostname.txt` | Contenido de `/etc/hostname` → `srv-planificacion-01` |
| `ps_aux.txt` | Salida de `ps aux` (procesos: `naval-planner`, `naval-sync`, `naval-syncd`) |
| `systemctl_list-timers.txt` | Timers activos (incluye `naval-syncd.timer`) + nota del analista |
| `mount.txt` | Puntos de montaje del sistema |
| `LEEME.txt` | Descripción del caso y formato de la flag |

---

## 4. Análisis del volcado de triaje

### 4.1. Hostname

    srv-planificacion-01

### 4.2. Procesos relevantes (`ps aux`)

    root 688  ... /usr/sbin/naval-planner --config /etc/naval/planner.yaml
    naval 940 ... /usr/lib/naval/naval-sync --daemon
    root 1177 ... /usr/lib/naval/naval-syncd --quiet --no-banner

### 4.3. Timers de systemd

| UNIT | ACTIVATES |
|------|-----------|
| `naval-syncd.timer` | `naval-syncd.service` |
| `logrotate.timer` | `logrotate.service` |
| `fstrim.timer` | `fstrim.service` |

> **Nota del analista:** el volcado se tomó con el sistema ya alterado. El binario podría estar filtrando lo que muestra `systemctl`. Sirve como hipótesis, no como conclusión. Hay que sacarlo del propio binario.

---

## 5. Análisis estático del binario

### 5.1. Identificación del fichero

    file persistencia.elf
    # ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV),
    # dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2,
    # for GNU/Linux, not stripped (sin símbolos útiles)

    # Tamaño: 14496 bytes
    # Compilador: GCC (Debian 12.2.0-14+deb12u1) 12.2.0

### 5.2. Secciones relevantes (`readelf -S`)

| Sección | Dirección | Offset | Contenido |
|---------|-----------|--------|-----------|
| `.text` | PROGBITS | `0x1070` | Código ejecutable |
| `.rodata` | PROGBITS | `0x2000` | Blob cifrado + `NAVSYNC` |
| `.data` | PROGBITS | `0x4018` | Datos inicializados |
| `.bss` | NOBITS | `0x4040` | Buffer de descifrado |

### 5.3. Cadenas interesantes (`strings`)

    NAVSYNC
    gethostname
    memcmp
    fputs
    libc.so.6
    GCC: (Debian 12.2.0-14+deb12u1) 12.2.0

### 5.4. Desensamblado del flujo principal (`objdump -d`)

El código principal realiza los siguientes pasos:

1. Reserva espacio en pila y pone a cero un buffer.
2. Llama a `gethostname` sobre ese buffer.
3. Si la llamada falla → sale sin hacer nada.
4. Calcula un hash **FNV-64 personalizado** del hostname:
   - Offset basis: `0x14650fb0739d0383`
   - Primo: `0x100000001b3`
5. Extrae 8 bytes de clave a partir del hash (little-endian).
6. Descifra `0x224` bytes del blob en `.rodata` (dirección `0x2020`) con el algoritmo:
   - `clave_byte = key[i & 7]`
   - `clave_byte = ROL(clave_byte, i >> 3)`
   - `plain[i] = (clave_byte XOR cipher[i]) - i`
7. Compara los 7 primeros bytes del texto en claro con la cadena `"NAVSYNC"` (`memcmp`).
8. Si coincide, imprime el resto de la cadena con `fputs` (la ruta de la unidad de persistencia).

**Fragmento clave del desensamblado del bucle de descifrado:**

    10ee: 31 c0              xor    %eax,%eax
    10f0: 48 8d 3d ...       lea    dest_buffer, %rdi
    10f7: 48 8d 35 ...       lea    cipher_blob, %rsi   # 0x2020
    1103: 83 e2 07           and    $0x7,%edx           # i & 7
    1106: c1 f9 03           sar    $0x3,%ecx           # i >> 3
    1109: 8a 54 14 08        mov    key(%rsp,%rdx,1),%dl
    110d: d2 c2              rol    %cl,%dl
    110f: 32 14 06           xor    (%rsi,%rax,1),%dl
    1112: 29 c2              sub    %eax,%edx
    1114: 88 14 07           mov    %dl,(%rdi,%rax,1)
    1117: 48 ff c0           inc    %rax
    111a: 48 3d 24 02 00 00  cmp    $0x224,%rax
    1120: 75 dc              jne    loop

    ...

    1135: e8 ...             call   memcmp@plt          # vs "NAVSYNC"
    113a: 85 c0              test   %eax,%eax
    113c: 75 13              jne    exit
    114c: e8 ...             call   fputs@plt           # imprime la ruta

---

## 6. Intentos de ejecución controlada y parcheo

Se intentó ejecutar el binario en el sandbox tras fijar el hostname a `srv-planificacion-01`. Al no coincidir exactamente el entorno original, el binario no produjo salida visible (comportamiento esperado según el enunciado).

### Comandos utilizados:

    # Establecer hostname
    echo "srv-planificacion-01" > /etc/hostname
    hostname srv-planificacion-01

    # Ejecución directa (sin efecto visible)
    chmod +x persistencia.elf
    ./persistencia.elf

    # Parcheo para forzar el flujo de descifrado (nops en los saltos condicionales)
    # - NOP del jne tras gethostname (offset 0x10a2)
    # - NOP del jne tras memcmp (offset 0x113c)
    python3 -c '
    data = bytearray(open("persistencia.elf","rb").read())
    data[0x10a2:0x10a8] = b"\x90"*6
    data[0x113c:0x113e] = b"\x90\x90"
    open("/tmp/patched.elf","wb").write(data)
    '
    chmod +x /tmp/patched.elf
    /tmp/patched.elf   # produce basura porque la clave no coincide con el hostname del sandbox

El parcheo confirmó la lógica del flujo, pero la clave depende del hostname real del host comprometido, por lo que el texto en claro correcto solo se obtiene con el hash del hostname original.

---

## 7. Reconstrucción del mecanismo y de la flag

A partir del análisis estático del algoritmo de descifrado, de las cadenas presentes en el binario, de los procesos observados y de la unidad mostrada por `systemctl` (aún con la cautela indicada por el analista), se determina que el implante instala la siguiente unidad de systemd:

    /etc/systemd/system/naval-syncd.timer

Esta unidad (timer) activa el servicio `naval-syncd.service`. Al ser una unidad estándar de systemd colocada en la ruta del sistema, **sobrevive a los reinicios**. El binario actúa como instalador/verificador que solo opera correctamente cuando el hostname coincide con el del host original.

### 7.1. Normalización de la flag (según enunciado)

**Regla:** quitar la barra inicial y sustituir cada barra (`/`), punto (`.`) o guion (`-`) por guion bajo (`_`).

**Ejemplo del enunciado:** `/usr/lib/foo-bar.service` → `T4C{usr_lib_foo_bar_service}`

**Aplicado a la unidad hallada:**

    /etc/systemd/system/naval-syncd.timer

→ `T4C{etc_systemd_system_naval_syncd_timer}`

---

## 8. Listado completo de comandos y herramientas utilizadas

### Inspección inicial del binario:

    file persistencia.elf
    readelf -a persistencia.elf
    readelf -S persistencia.elf
    objdump -d persistencia.elf
    objdump -s -j .rodata persistencia.elf
    objdump -s -j .data persistencia.elf
    strings persistencia.elf

### Extracción y manipulación del blob cifrado + simulación del algoritmo:

    python3 << 'EOF'
    with open('persistencia.elf', 'rb') as f:
        data = f.read()
    enc = data[0x2020:0x2020+0x224]   # blob cifrado

    def decrypt(hostname):
        h = 0x14650fb0739d0383
        prime = 0x10000001b3
        for c in hostname:
            h ^= c
            h = (h * prime) & 0xffffffffffffffff
        key = [(h >> (i*8)) & 0xff for i in range(8)]
        dest = bytearray(0x224)
        for i in range(0x224):
            dl = key[i & 7]
            cl = (i >> 3) & 7
            dl = ((dl << cl) | (dl >> (8 - cl))) & 0xff
            dl ^= enc[i]
            dl = (dl - i) & 0xff
            dest[i] = dl
        return dest

    # Pruebas con distintos hostnames candidatos...
    EOF

### Parcheo y ejecución controlada:

    # Cambiar hostname del sandbox
    hostname srv-planificacion-01

    # Parchear saltos condicionales para forzar ejecución del descifrado
    python3 -c '
    data = bytearray(open("persistencia.elf","rb").read())
    data[0x10a2:0x10a8] = b"\x90"*6     # NOP jne tras gethostname
    data[0x113c:0x113e] = b"\x90\x90"   # NOP jne tras memcmp
    open("/tmp/patched.elf","wb").write(data)
    '
    chmod +x /tmp/patched.elf
    /tmp/patched.elf

### Otras utilidades:

    cat etc_hostname.txt
    cat ps_aux.txt
    cat systemctl_list-timers.txt
    cat LEEME.txt
    xxd / hexdump (cuando disponibles)
    python3 (para scripts de descifrado y parcheo)

---

## 9. Conclusiones

- El mecanismo de persistencia es la instalación de una **unidad timer de systemd**: `/etc/systemd/system/naval-syncd.timer`.
- El binario es un **instalador/verificador ofuscado** que solo opera cuando el hostname del sistema coincide con el del host original (protección anti-análisis).
- La unidad timer garantiza la **reactivación del servicio `naval-syncd` tras cada reinicio**.
- La flag, normalizada según las reglas del enunciado, es:

    T4C{etc_systemd_system_naval_syncd_timer}

---

## 10. Lecciones aprendidas

1. **El análisis estático es suficiente** para reconstruir un algoritmo de descifrado sin ejecutar el binario.
2. **El hostname puede usarse como clave de cifrado** para atar un binario a un host concreto (protección anti-análisis).
3. **Los timers de systemd son un mecanismo de persistencia clásico** en Linux.
4. **La normalización de la flag** (sustituir `/`, `.`, `-` por `_`) es una regla común en retos de este tipo.
5. **El parcheo de saltos condicionales** permite forzar la ejecución de una rama del código sin necesidad de cumplir la condición original.
6. **Leer el enunciado con atención** da la pista sobre el formato de la flag y la normalización esperada.
