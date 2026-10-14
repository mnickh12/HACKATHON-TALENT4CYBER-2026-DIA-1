# Rastro del Ingeniero — PIR-03

## 1. Datos del reto

| Campo | Valor |
|-------|-------|
| **Nombre** | PIR-03. ¿Cómo obtuvo el adversario una credencial válida? |
| **Categoría** | OSINT |
| **Objetivo** | Determinar cómo el adversario obtuvo una credencial válida sin explotar ninguna vulnerabilidad |
| **Formato de entrega** | `T4C{...}` |
| **Flag** | `T4C{AKIA5RJQ2LNVXW3PD7TG}` |

---

## 2. Resumen y entrega

El adversario obtuvo una credencial válida de **Rubén Belmonte**, ingeniero de integraciones de Maritime Logistics Iberia, que la dejó expuesta en el **historial de un repositorio público**.

La entrega es:

    T4C{AKIA5RJQ2LNVXW3PD7TG}

**Pregunta del reto:** ¿cómo obtuvo el adversario una credencial válida?

El acceso inicial **no requirió explotar ninguna vulnerabilidad**. El EEI pedido es el **identificador de la credencial**, no el secreto.

---

## 3. Material y metodología

El material es un volcado estático de fuentes abiertas: tres perfiles públicos y un repositorio git. **No se hizo ningún reconocimiento contra sistemas reales.**

| Fuente | Qué aporta |
|--------|------------|
| `LEEME.txt` | Hipótesis (credencial válida expuesta) y aviso: el repositorio está limpio ahora, pero no siempre lo estuvo |
| Foro PortuLogistics | Usuario `rbelmonte-dev`, cargo y mensaje del 2025-11-03 |
| Perfil en `git.example` | Un único repositorio público, `mli-tracking-tools` |
| Red profesional | Nombre real, empresa y correo corporativo |
| `mli-tracking-tools.bundle` | Historial git completo, restaurado con `git clone` |

**Método:** primero se identificó a la persona cruzando los perfiles, y después se revisó con `git log -p` el **historial completo** del repositorio, no solo su último estado.

---

## 4. Cadena de evidencias

### 4.1. Identificación

La red profesional muestra a **Rubén Belmonte**, ingeniero de integraciones en Maritime Logistics Iberia (Algeciras), con correo `rbelmonte@mli-iberia.example`. Lista como otro perfil suyo `rbelmonte-dev`.

### 4.2. Enlace con el foro

El foro muestra `rbelmonte-dev` como "Rubén B.", con el mismo cargo y empresa. El **2025-11-03** escribió que su script está en el repo público `mli-tracking-tools` y que *"la parte de credenciales la tuve que limpiar luego"*. Avisó de no copiar el `settings` del primer commit.

### 4.3. Repositorio

`git.example` confirma que `rbelmonte-dev` tiene un solo repositorio público, `mli-tracking-tools`, con último commit el **2025-11-07**. Los commits están firmados por R. Belmonte.

### 4.4. Fuga

En el estado actual, `config/settings.yaml` solo remite a variables de entorno. El historial cuenta otra cosa:

| Commit | Fecha | Efecto |
|--------|-------|--------|
| `1801590` | 2025-11-03 | Crea `settings.yaml` con `access_key_id` y `secret_access_key` en claro, más `usuario_servicio` |
| `54c73ca` | 2025-11-04 | Cambia la región a `eu-south-2`; la credencial sigue en el archivo |
| `a83d187` | 2025-11-06 | Elimina las credenciales del archivo, pero siguen en el historial |

### 4.5. Alcance

La clave da acceso al bucket `mli-manifiestos-archivo`, el almacén de manifiestos que sincroniza el script.

El identificador filtrado es `AKIA5RJQ2LNVXW3PD7TG`. Este informe **no reproduce el secreto**.

---

## 5. Conclusiones y recomendaciones

La credencial **no se obtuvo por explotación**, sino por **exposición involuntaria**: un secreto subido en el primer commit y "limpiado" tres días después sigue íntegro en el historial público. Eliminarlo del último estado **no lo retira de la fuente**.

### Recomendaciones:

- **Revocar y rotar de inmediato** la clave `AKIA5RJQ2LNVXW3PD7TG`, y revisar el registro de accesos al bucket desde el 2025-11-03.
- **Reescribir el historial del repositorio o retirarlo**, sabiendo que cualquier copia previa (como el mirror del foro) ya contiene el secreto.
- **Rotar también cualquier credencial asociada al usuario de servicio** `svc-tracking`, que aparece en el primer commit.
- **Aplicar detección de secretos antes de cada commit** y credenciales de corta duración con permisos mínimos.

---

## 6. Lecciones aprendidas

- **El historial de Git es público y permanente.** Eliminar un secreto del último commit no lo elimina del historial.
- **Los secretos en repositorios públicos son una fuga inmediata.** Hay que usar variables de entorno o gestores de secretos.
- **La detección de secretos antes del commit** (pre-commit hooks, `git-secrets`, `trufflehog`) es esencial.
- **Las credenciales de corta duración** con permisos mínimos reducen el impacto de una fuga.
- **El OSINT puede revelar mucho** cruzando perfiles públicos y repositorios.
