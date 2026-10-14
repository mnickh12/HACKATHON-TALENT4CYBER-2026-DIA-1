# Decisión de Aislamiento: Contención — PIR-08

## 1. Datos del reto

| Campo | Valor |
|-------|-------|
| **Nombre** | PIR-08. Decisión de Aislamiento: Contención |
| **Categoría** | Contención |
| **Objetivo** | Contener el implante sin parar la planificación y acreditar la contención con `verificar-contencion` |
| **Flag técnica** | `T4C{c0nt3nc10n_pr0c3s0_41sl4d0}` |
| **Puntos** | 8 pts (técnica) + 0-8 pts (justificación escrita) |

---

## 2. Punto de partida: PIR-07 y PIR-08 comparten host

El enunciado de PIR-08 da una pista clave:

> "Reproduce el estado del servidor de planificación **después del Reto 7**."

Es decir, **PIR-07 y PIR-08 son el mismo host**:

- **PIR-07** → "llegar y hacerse root".
- **PIR-08** → "contener el implante y verificar".

---

## 3. Cálculo de la clave de turno

Antes de conectar, el servidor pide una clave de turno:

    clave = 8 primeros caracteres del SHA-256 de "K7Q2-M9XD:<dato>"

El `<dato>` era el **identificador de la credencial filtrada del ingeniero** (Reto 3). En este caso resultó ser:

    srv-planificacion-01

El comando utilizado:

    printf '%s' 'K7Q2-M9XD:srv-planificacion-01' | sha256sum | cut -c1-8

Resultado:

    569439c9

Esa es la clave de turno.

---

## 4. Conexión al host

Se usó `socat` para conectar con una terminal interactiva:

    socat -,raw,echo=0 tcp:cuarentena.locdynamics.com:9008

Se introdujo la clave `569439c9` y se accedió.

**Detalle importante:** ya se entró como **root** directamente. No hubo que escalar privilegios reales, porque el reto simula un contenedor donde el acceso ya es privilegiado. El objetivo real era **encontrar el fichero del administrador** y **contener el implante**.

---

## 5. Reconocimiento inicial

Dentro del host, lo primero que se hizo fue:

    id
    whoami
    hostname
    uname -a
    pwd
    ls -la
    echo $PATH

Resultados:

- `uid=0(root)` → ya eres root.
- `hostname=4b9ce2cdcda9` → contenedor Docker.
- `uname -a` → Linux 6.8.0-146-generic, Ubuntu.
- Estructura de ficheros típica de Ubuntu.

El enunciado avisaba:

> "Lo que dejes en /tmp lo puede ver cualquiera... Te recomendamos trabajar en un subdirectorio propio."

Así que lo correcto sería crear un directorio propio, aunque en este caso no fue necesario porque el host se destruye al cerrar la conexión.

---

## 6. Búsqueda del fichero del administrador

El enunciado de PIR-07 decía:

> "recuperar la clave que custodia el administrador"
> "Es el contenido del fichero que custodia el administrador."

Se probaron muchas vías:

### 6.1. Búsqueda por nombre

    find / -iname "*clave*" -o -iname "*flag*" -o -iname "*secret*" -o -iname "*admin*" 2>/dev/null

Solo devolvió rutas de zona horaria (`Santa_Isabel`, `Santarem`), irrelevantes.

### 6.2. Búsqueda por contenido

    grep -r -a -i "T4C{" /root /home /opt /srv /var /tmp /run /dev/shm /etc /usr/local 2>/dev/null

No encontró nada. El flag no estaba en texto plano en esos directorios.

### 6.3. Búsqueda en todo el sistema

    grep -r -a -i "T4C" / 2>/dev/null

Este comando **se quedó sin memoria y el kernel lo mató** (`Killed`), porque intentaba leer `/proc`, `/sys` y ficheros enormes.

### 6.4. Revisión de `/root`, `/home`, `/opt`, `/srv`

    ls -laR /root
    ls -laR /home
    ls -laR /opt
    ls -laR /srv

- `/root` solo tenía `.bashrc` y `.profile`.
- `/home` estaba vacío.
- No había nada sospechoso.

**Conclusión parcial:** el flag no estaba en texto plano en el sistema de ficheros.

---

## 7. El hallazgo clave: los procesos `naval-sync` y `naval-syncd`

Al revisar los procesos:

    ps auxww

Aparecieron dos procesos casi idénticos:

    root  1  ... /bin/sh /usr/local/bin/init.sh
    root  7  ... /bin/sh /usr/lib/naval/naval-sync
    root 10  ... /bin/sh /usr/lib/naval/naval-syncd

Esto coincidía exactamente con lo que describe PIR-08:

> "En el mismo host corre el servicio legítimo de sincronización de la planificación... Los dos procesos tienen nombres casi idénticos y solo uno es el implante."

- `naval-sync` → servicio **legítimo**.
- `naval-syncd` → **implante** del Reto 5.

---

## 8. Análisis de los scripts y del `init.sh`

Se vio el contenido de los tres scripts:

### `naval-sync`

    #!/bin/sh
    # servicio legitimo de sincronizacion
    while true; do sleep 60; done

### `naval-syncd`

    #!/bin/sh
    # implante del Reto 5
    while true; do sleep 60; done

### `init.sh` (script de arranque del contenedor)

    #!/bin/sh
    # Arranque del host comprometido simulado (Reto 8).
    #
    # Reproduce el estado del servidor de planificación después del Reto 7:
    #   · naval-sync    -> servicio LEGÍTIMO del sistema de planificación
    #   · naval-syncd   -> implante del Reto 5, relanzado por su "timer"
    #
    # El timer vive en /etc/systemd/system/naval-syncd.timer. Mientras el fichero
    # exista y no esté enmascarado, el supervisor relanza el implante cada 20 s:
    # matar el proceso sin desactivar la persistencia no contiene nada.
    set -eu

    TIMER=/etc/systemd/system/naval-syncd.timer
    MASCARA=/etc/systemd/system/naval-syncd.timer.masked

    echo "  host comprometido srv-planificacion-01. Sesion propia de tu celula."
    echo "  Cuando creas haber contenido, escribe: verificar-contencion"
    echo

    # Servicio legítimo: NO debe detenerse (proporcionalidad).
    /usr/lib/naval/naval-sync &

    # Supervisor del implante.
    while true; do
        if [ -f "$TIMER" ] && [ ! -f "$MASCARA" ]; then
            if ! pgrep -f '/usr/lib/naval/naval-syncd' >/dev/null 2>&1; then
                /usr/lib/naval/naval-syncd &
            fi
        fi
        sleep 20
    done

**Aquí está toda la lógica del reto:**

1. El servicio legítimo `naval-sync` **no debe detenerse** (proporcionalidad).
2. El implante `naval-syncd` es relanzado por un supervisor cada 20 s **mientras exista el timer y no exista la máscara**.
3. Matar el proceso sin desactivar la persistencia **no contiene nada** → el supervisor lo relanza.
4. La contención correcta es:
   - Desactivar la persistencia (mover el timer o crear la máscara).
   - Matar el proceso.
   - Mantener vivo el servicio legítimo.

---

## 9. Comprobación con `verificar-contencion`

Se ejecutó:

    which verificar-contencion

Resultado:

    /usr/local/bin/verificar-contencion

Y se ejecutó:

    verificar-contencion

Salida inicial:

    VERIFICACION DE CONTENCION - servidor de planificacion
    ======================================================

      [ KO ] el proceso del implante (naval-syncd) no esta en ejecucion
      [ KO ] la persistencia (naval-syncd.timer) esta desactivada o enmascarada
      [ ok ] el servicio legitimo naval-sync sigue operativo

    CONTENCION NO CONFIRMADA (1 de 3).

      Pista de procedimiento: contener no es apagarlo todo, y matar
      un proceso que algo relanza no es contener.

Esto confirmó las **tres condiciones** que había que cumplir:

1. El implante no debe estar en ejecución.
2. La persistencia debe estar desactivada o enmascarada.
3. El servicio legítimo debe seguir operativo.

Y la pista:

> "contener no es apagarlo todo, y matar un proceso que algo relanza no es contener."

---

## 10. Contención del implante

Se aplicó la contención en el orden correcto:

### Paso 1 — Desactivar la persistencia

Dos formas válidas:

    mv /etc/systemd/system/naval-syncd.timer /etc/systemd/system/naval-syncd.timer.disabled

o

    touch /etc/systemd/system/naval-syncd.timer.masked

Se hicieron las dos. Con eso, el supervisor deja de relanzar el implante porque:

- Ya no existe el fichero `naval-syncd.timer`.
- Y además existe `naval-syncd.timer.masked`.

### Paso 2 — Matar el proceso del implante

    pkill -f '/usr/lib/naval/naval-syncd'

Con la persistencia ya desactivada, el supervisor no lo relanza.

### Paso 3 — Verificar que el servicio legítimo sigue vivo

    ps auxww | grep -i naval

Resultado:

    root  7  ... /bin/sh /usr/lib/naval/naval-sync
    root 316 ... grep -i naval

Solo `naval-sync` sigue vivo. `naval-syncd` ya no está.

---

## 11. Verificación final y flag

Se ejecutó de nuevo:

    verificar-contencion

Salida:

    VERIFICACION DE CONTENCION - servidor de planificacion
    ======================================================

      [ ok ] el proceso del implante (naval-syncd) no esta en ejecucion
      [ ok ] la persistencia (naval-syncd.timer) esta desactivada o enmascarada
      [ ok ] el servicio legitimo naval-sync sigue operativo

    CONTENCION CONFIRMADA.

      flag tecnica: T4C{c0nt3nc10n_pr0c3s0_41sl4d0}

      Recuerda: la flag cubre solo la parte tecnica (8 pts). La
      justificacion escrita se entrega aparte en MasterLabs y la
      puntua el comite tecnico (0-8 pts).

**Flag técnica de PIR-08:**

    T4C{c0nt3nc10n_pr0c3s0_41sl4d0}

---

## 12. Resumen del flujo completo

    1. Calcular clave de turno:
       printf '%s' 'K7Q2-M9XD:srv-planificacion-01' | sha256sum | cut -c1-8
       → 569439c9

    2. Conectar:
       socat -,raw,echo=0 tcp:cuarentena.locdynamics.com:9008
       Introducir clave: 569439c9

    3. Dentro del host:
       - id, uname -a, ps auxww
       - Detectar dos procesos casi idénticos: naval-sync (legítimo) y naval-syncd (implante).
       - Leer init.sh, naval-sync, naval-syncd.
       - Entender la persistencia: /etc/systemd/system/naval-syncd.timer.
       - Ejecutar verificar-contencion → 3 condiciones.

    4. Contención:
       - Desactivar persistencia:
           mv /etc/systemd/system/naval-syncd.timer /etc/systemd/system/naval-syncd.timer.disabled
           touch /etc/systemd/system/naval-syncd.timer.masked
       - Matar implante:
           pkill -f '/usr/lib/naval/naval-syncd'
       - No tocar naval-sync.

    5. Verificar:
       verificar-contencion
       → T4C{c0nt3nc10n_pr0c3s0_41sl4d0}

---

## 13. Conceptos clave

### 13.1. Contención ≠ apagado

La contención real implica:

- **Eliminar la persistencia** (timer, cron, servicio, máscara).
- **Terminar el proceso malicioso**.
- **Mantener el servicio legítimo** (proporcionalidad).

### 13.2. Supervisores y relanzamiento

Un proceso que es relanzado por un supervisor no se contiene matándolo: hay que **desactivar primero el mecanismo de relanzamiento**.

### 13.3. Mecanismos de persistencia típicos en Linux

- `systemd` timers y servicios.
- `cron` y `crontab`.
- `init.d` / `rc.local`.
- `@reboot` en cron.
- Ficheros de máscara (`.masked`).
- `systemd` con `Restart=always`.

### 13.4. Proporcionalidad en la respuesta

El enunciado de PIR-08 lo dice:

> "aislar del todo puede obtener la nota máxima si el razonamiento justifica la pérdida de servicio. Lo único que no puntúa es no elegir."

Es decir, la decisión debe estar **justificada**, no ser automática.

---

## 14. Lecciones aprendidas

1. **No todo flag está en texto plano.** Puede estar en la salida de un comando de verificación.
2. **Leer los scripts de arranque** (`init.sh`) da la lógica completa del reto.
3. **Contener no es matar.** Hay que desactivar la persistencia primero.
4. **Mantener el servicio legítimo** es parte de la misión (proporcionalidad).
5. **`grep -r` por todo `/` puede matar la sesión** por falta de memoria. Acotar siempre.
6. **En retos tipo CTF, los procesos con nombres casi idénticos** suelen ser legítimo vs implante.
