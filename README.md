# HACKATHON TALENT4CYBER 2026: DÍA 1

![Talent4Cyber](https://img.shields.io/badge/Talent4Cyber-2026-blue)
![Día](https://img.shields.io/badge/D%C3%ADa-1%20de%202-orange)
![MCCE](https://img.shields.io/badge/MCCE-Ministerio%20de%20Defensa-green)

## Sobre el evento

**Talent4Cyber 2026** es la tercera edición del primer **Attack & Defense Hackathon universitario nacional de ciberseguridad**, una iniciativa del **Ministerio de Defensa de España**, a través del **Mando Conjunto del Ciberespacio (MCCE)**, en colaboración con la **Universidad de Granada**, la **Universidad Politécnica de Cartagena** y **Trustlab**.

El objetivo es acercar a la comunidad universitaria a la misión de los **Guardianes del Ciberespacio**. Durante dos días, los mejores talentos universitarios se enfrentan a desafíos técnicos en un escenario donde la **guerra electrónica** y las **infraestructuras marítimas** marcan la diferencia.

**Misión:** Defender el tráfico marítimo del Estrecho y sus infraestructuras esenciales para la defensa mediante estrategias de ataque y defensa en tiempo real.

- **Fechas:** 14 y 15 de octubre de 2026
- **Horario:** Día 1: de 9:00 a 17:00
- **Formato:** Attack & Defense (KotH)
- **Participación:** Equipos de 4 personas de la misma universidad
- **Premio:** Participar en **Locked Shields 2027**, organizado por el **CCDCOE** (OTAN)

Más información: [https://talent4cyber.es/](https://talent4cyber.es/)

---

## Datos del equipo

| Campo | Valor |
|-------|-------|
| **Nombre del equipo** | Brigada Bardino |
| **Universidad** | Universidad Europea de Madrid |
| **Miembro** | marcos |
| **Puntos completados (individual)** | 95 |
| **Puntos totales del equipo** | 123 |
| **Tiempo en completarlo** | 5h 56m 40s 96ms |

---

## Operación Estrecho Fantasma

> *"03:31Z. Un buque a 361 nudos. Ningún barco del mundo hace eso. Pero ahí está, en las pantallas del control de tráfico marítimo de Algeciras. Rumbo 044. Doce millas náuticas en dos minutos. Sin eco de radar que lo acompañe y sin figurar en ningún registro de buques autorizados.*
>
> *Esa misma madrugada, alguien ha bajado la potencia del cable submarino que conecta dos continentes. No lo ha cortado: lo ha degradado. Un corte se ve. Una degradación se discute.*
>
> *Por el Estrecho de Gibraltar pasan hasta trescientos buques al día y un ferry cargado de pasaje cada media hora. Todos se fían de unos instrumentos que, desde esa noche, mienten.*
>
> *Tienes dos días para averiguar quién y cómo."*

### Día 1: Reconstruir

Un manifiesto cifrado, un portal con una puerta trasera, un ingeniero que subió más de la cuenta a un repositorio público, un correo con carga y un implante que sobrevive a los reinicios.

**Criptografía, OSINT, forense, ingeniería inversa y más.**

---

## Retos resueltos (Día 1)

| # | Reto | Categoría | Puntos | Flag / Solución |
|---|------|-----------|--------|------------------|
| 01 | [Puerta Trasera del Proveedor](Puerta%20Trasera%20del%20Proveedor.md) | Web | 10 | `T4C{jwt_4lg_c0nfus10n_RS256_HS256}` |
| 02 | [Manifiesto Cifrado](Manifiesto%20Cifrado.md) | Criptografía | 10 | `T4C{f4ct0r_c0mun_ENV_4LG_77413}` |
| 03 | [Correo con Carga](Correo%20con%20Carga.md) | Forense | 10 | `T4C{m4cr0_st4g3r_b4s364_0culto}` |
| 04 | [Ruta Desviada](Ruta%20Desviada.md) | Scripting | 10 | `T4C{gr4f0_h4sh_p4rc14l_ru74_d3sv14d4}` |
| 05 | [Decisión de Aislamiento: Contención](Decisión%20de%20Aislamiento%20-%20Contención.md) | Contención | 8 | `T4C{c0nt3nc10n_pr0c3s0_41sl4d0}` |
| 06 | [Decisión de Aislamiento: Justificación](Decisión%20de%20Aislamiento%20-%20Justificación.md) | Contención | 0-8 | *(Justificación escrita)* |
| 07 | [Escalada al Núcleo](Escalada%20al%20Núcleo.md) | Pwn | 12 | `T4C{p4th_h1j4ck_SU1D_nucl30_r00t}` |
| 08 | [Bloqueo del Sistema](Bloqueo%20del%20Sistema.md) | Pwn | 15 | `T4C{pyj41l_3sc4p3_subcl4ss3s_0k}` |
| 09 | [Contenedor Fantasma](Contenedor%20Fantasma.md) | Criptografía | 15 | `T4C{m3rs3nn3_tw1st3r_s33d_d3b1l}` |
| 10 | [Rastro del Ingeniero](Rastro%20del%20Ingeniero.md) | OSINT | 8 | `T4C{AKIA5RJQ2LNVXW3PD7TG}` |
| 11 | [Interceptación Aérea](Interceptación%20Aérea.md) | Tareas complementarias | 0 | `INTOXICACIÓN CONFIRMADA` |
| 12 | [Persistencia Silenciosa](Persistencia%20Silenciosa.md) | Reversing | 12 | `T4C{etc_systemd_system_naval_syncd_timer}` |

---

## Metodología

Todo el análisis se realizó en **entorno controlado** y de forma **estática** siempre que fue posible:

- **Criptografía:** Ataques de factorización (GCD), confusión de algoritmos (JWT), PRNG débiles.
- **OSINT:** Análisis de perfiles públicos, historial de Git, credenciales expuestas.
- **Forense:** Análisis de cabeceras de correo, extracción de macros VBA, desofuscación.
- **Reversing:** Análisis estático de binarios ELF, parcheo de saltos condicionales.
- **Pwn:** PATH hijacking en binarios SUID, escape de PyJail.
- **Web:** IDOR, BOLA, Algorithm Confusion Attack.
- **Contención:** Análisis de persistencia en systemd, contención proporcional.

---

## Notas

- Este repositorio recoge los **apuntes y soluciones** del Día 1 de la hackathon.
- El **Día 2** se centró en la parte de **decisión y respuesta** (AIS, firmware, comunicados, ROE).
- No todo el material recibido era útil: parte estaba diseñado para **perder el tiempo**. Saber cuándo cerrar una línea de investigación también puntúa.
- Los retos se resolvieron en equipo, pero este repositorio es una recopilación personal para estudio y referencia.

---

## Autor

**marcos** (mnickh12)

- GitHub: [@mnickh12](https://github.com/mnickh12)

---

## Licencia

Este proyecto es solo para **fines educativos**. Las flags y soluciones son parte de una competición pasada y no deben usarse para actividades ilegales.

---

> *"La ciberdefensa no se gana en solitario. En el mundo real, las operaciones de defensa y ataque requieren trabajo en equipo, coordinación y diversidad de habilidades."*
