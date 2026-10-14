# Puerta Trasera del Proveedor — PIR-02

## 1. Datos del reto

| Campo | Valor |
|-------|-------|
| **Nombre** | PIR-02. ¿Por dónde se movió el adversario dentro del proveedor? |
| **Plataforma** | Maritime Logistics Iberia (MLI Connect) |
| **URL base** | `https://puerto.locdynamics.com/reto02/` |
| **Credenciales iniciales** | Usuario: `cliente` / Contraseña: `cliente2026` |
| **Objetivo** | Escalar desde cuenta de cliente hasta el panel de gestión interna y extraer la clave de conciliación de la incidencia `INC-0104` |
| **Formato de entrega** | `T4C{...}` |
| **Pista del enunciado** | "El reto se resuelve con la cabecera Authorization y el inicio de sesión usaría esa misma cabecera. No hay nada que descargar." |

---

## 2. Resultado final

**Flag obtenida:**

```text
T4C{jwt_4lg_c0nfus10n_RS256_HS256}
