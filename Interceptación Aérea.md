# Interceptación Aérea

## 1. Datos del reto

| Campo | Valor |
|-------|-------|
| **Nombre** | Interceptación Aérea |
| **Categoría** | Tareas complementarias |
| **Referencia** | INT-2026-0211-A |
| **Objetivo** | Analizar las muestras de RF y determinar si contienen inteligencia explotable |
| **Formato de entrega** | `INTOXICACIÓN CONFIRMADA` |
| **Resultado** | `INTOXICACIÓN CONFIRMADA` |

---

## 2. Resumen y entrega

El material **no aporta inteligencia explotable**: es ruido sintético uniforme, sin señal. La entrega es:

    INTOXICACIÓN CONFIRMADA

**Contexto:** contacto aéreo no cooperativo, presunto UAS de clase I, sobre el corredor norte del Estrecho (2026-02-11, 22:41Z). Hay 12 s de grabación en banda ISM de 2,4 GHz, y el equipo de captura no pudo confirmar modulación ni preámbulo. El procedimiento permite declarar la **esterilidad del material** y liberar esfuerzo hacia otras líneas.

---

## 3. Material y metodología

El parte de interceptación adjunta dos archivos con las mismas muestras:

- Un `.wav` (IQ desentrelazado, estéreo de 16 bits, 48 kHz; I a la izquierda, Q a la derecha).
- Un `.iq` en crudo (int16 intercalado I,Q).

Se comprobó que **ambos son idénticos**: 1.152.000 valores, 576.000 muestras complejas, 12 s.

Como el material llega sin preprocesar, se partió de contrastar la hipótesis de que hay una señal y se buscó estructura de **cuatro formas**:

1. La distribución de amplitudes.
2. La autocorrelación (para ver símbolos o preámbulo).
3. El espectro.
4. La evolución de la potencia.

El análisis fue **local**, con Python y `numpy`.

---

## 4. Resultados del análisis

| Prueba | Resultado | Lectura |
|--------|-----------|---------|
| **Amplitud** | I y Q entre -12000 y +12000 exactos; curtosis 1,80 | Distribución uniforme con tope duro. Una captura RF real sería gaussiana, sin tope |
| **Autocorrelación** | Máximo de 0,004 en I y 0,010 en la magnitud, fuera del lag cero | Sin periodicidad de símbolo, preámbulo ni repetición |
| **Espectro** | Plano; pico 17 veces la mediana | Lo esperable de ruido blanco con unos 131.000 bins |
| **Correlación I-Q** | 0,0006 | Sin modulación que los acople |
| **Signo de I** | 49,9 % de unos; 50,0 % de transiciones | Comportamiento de bits aleatorios |
| **Potencia por segundo** | Constante, entre 94,5 y 96,5 millones | Sin desvanecimiento ni ráfagas, a pesar de que el parte dice que el contacto se pierde a los pocos segundos |

---

## 5. Conclusión

Las seis pruebas apuntan a lo mismo: el archivo es **ruido uniforme generado por software**, no una captura de un enlace de dron. No hay modulación, preámbulo ni comportamiento temporal que explotar.

Se declara **INTOXICACIÓN CONFIRMADA** y el esfuerzo pasa a otras líneas.

**Limitación:** esto concluye sobre estas muestras y no sobre la existencia del contacto aéreo, que debe seguir verificándose por otras fuentes.

---

## 6. Lecciones aprendidas

- **No todo material de interceptación contiene señal.** El ruido sintético es indistinguible de una captura real si no se analiza en profundidad.
- **La distribución de amplitudes es una prueba clave.** Una captura RF real sigue una distribución gaussiana; una generada por software suele tener un tope duro.
- **La autocorrelación revela periodicidad.** Si no hay picos fuera del lag cero, no hay símbolos ni preámbulo.
- **El espectro plano es típico de ruido blanco.** Un pico anómalo puede indicar una señal, pero aquí solo había ruido.
- **La correlación I-Q nula indica ausencia de modulación.** En una señal real, I y Q suelen estar acoplados.
- **La potencia constante descarta ráfagas o desvanecimientos.** Si el parte dice que el contacto se pierde, pero la potencia no varía, algo no cuadra.
