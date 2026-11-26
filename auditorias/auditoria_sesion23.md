# Bitácora de Requerimientos y Auditoría IA — Sesión 23
**Materia:** Lenguajes y Autómatas II (SCD-1016)  
**Tema:** Etiquetas y Saltos Incondicionales (Goto)  
**Docente Titular:** MCC. Iván Márquez Larios (`ijmarquezl`)  
**Estudiante:** Diego Reyes Martínez  
**Matrícula:** 24531376  
**Track:** C (el harness evalúa C; la versión en Rust es opcional y no se califica con este harness)  

---

## 1. Fase 1: El Artesano (Dificultad Manual)
*Reto:* Generar etiquetas únicas y escribir el GOTO o la marca de destino  
*Describe qué problema o restricción de memoria/sintaxis identificaste al codificar a mano:*
> Esta fue rápida porque `nueva_etiqueta` es casi igual a `nuevo_temporal` de la sesión 21, nada más cambia `_t` por `L`. Lo del post-incremento ya lo tenía aprendido: primero `snprintf` con el valor y luego `(*contador)++`, con paréntesis.
>
> En `emitir_flujo` me fijé en dos detalles del formato que se me podían ir: `GOTO` lleva un espacio antes de la etiqueta, y la marca lleva los dos puntos **pegados** (`L2:`, no `L2 :`). Como el harness compara exacto, un espacio de más ya da FALLA.
>
> Al principio puse `if (es_salto == 1)`, pero luego leí que el contrato dice "distinto de cero", así que lo cambié a `!= 0`. En C cualquier número que no sea 0 es verdadero.
>
> Para el NULL reviso primero `etiqueta == NULL` y pongo `dst[0] = '\0'` para que quede vacía, si no el `snprintf` con `%s` y NULL puede tronar.

---

## 2. Fase 2: El Científico (Ingeniería de Requerimientos C-R-E-O)

### 2.1 Matriz de Especificación Formal
| Elemento C-R-E-O | Definición Técnica |
| :--- | :--- |
| **Contexto (C)** | C99 / GCC o Rust 2021 sin bibliotecas no estándar |
| **Rol (R)** | Ingeniero de software de sistemas y compiladores |
| **Especificación (E)** | Etiqueta `LK` con post-incremento, `GOTO Lx` si es_salto ≠ 0 y `Lx:` si es 0 |
| **Output (O)** | Solo código modular y análisis de complejidad |

### 2.2 Prompt Estructurado
```text
CONTEXTO: Laboratorio de compiladores en C (C11, GCC) sobre etiquetas y saltos
incondicionales en código intermedio. Se compila con gcc -std=c11 -Wall -Wextra
-Werror, sin warnings. Solo librerías estándar, sin malloc, sin globales y sin
main() porque el harness ya tiene uno. Las funciones escriben en un buffer.

ROL: Actúa como ingeniero de software de sistemas y compiladores.

ESPECIFICACIÓN: Respeta exactamente estas firmas de api_sesion23.h:

void nueva_etiqueta(char *dst, int cap, int *contador);
void emitir_flujo(char *dst, int cap, const char *etiqueta, int es_salto);

Reglas:
- nueva_etiqueta escribe "L" + valor actual del contador y DESPUÉS lo
  incrementa. Contador 0 -> "L0" y queda en 1.
- emitir_flujo: si es_salto es DISTINTO DE CERO (1, 2, -1...) escribe
  "GOTO etiqueta"; si es 0 escribe "etiqueta:" (dos puntos pegados).
- etiqueta NULL produce cadena vacía sin tronar.
- Usa snprintf con cap; si dst es NULL o cap <= 0 no hagas nada.

Ejemplos: "L0", "L5", "L10", "GOTO L2", "L2:", "".

OUTPUT: Solo el código de ia_sesion23_c.c, corto, y una línea con la complejidad.
```

---

## 3. Código Devuelto por el Modelo
```c
#include "api_sesion23.h"
#include <stdio.h>

void nueva_etiqueta(char *dst, int cap, int *contador) {
    if (!dst || cap <= 0) return;
    if (!contador) { dst[0] = '\0'; return; }
    snprintf(dst, (size_t)cap, "L%d", *contador);
    *contador += 1;   // post-incremento: primero L0
}

void emitir_flujo(char *dst, int cap, const char *etiqueta, int es_salto) {
    if (!dst || cap <= 0) return;
    if (!etiqueta) { dst[0] = '\0'; return; }
    // cualquier valor distinto de 0 cuenta como salto
    snprintf(dst, (size_t)cap, es_salto ? "GOTO %s" : "%s:", etiqueta);
}

// Complejidad: O(L) en las dos, L = largo del texto (limitado por cap)
```

---

## 4. Fase 3: El Auditor (Resultados del Test Harness)
*Pruebas ejecutadas:* Etiquetas L0, L5 y L10, GOTO, marca de destino y etiqueta NULL

### Salida del harness
```text
=== AUDITORIA SESION 23 - Etiquetas y saltos incondicionales ===
Caso                               esperado:              obtenido:              resultado
[ 1] nueva_etiqueta(L0)                 esperado: L0           obtenido: L0           OK
[ 2] nueva_etiqueta(L5)                 esperado: L5           obtenido: L5           OK
[ 3] emitir_flujo(GOTO L2)              esperado: GOTO L2      obtenido: GOTO L2      OK
[ 4] emitir_flujo(marca L2)             esperado: L2:          obtenido: L2:          OK
[ 5] emitir_flujo(NULL)                 esperado:              obtenido:              OK
[ 6] nueva_etiqueta(L10)                esperado: L10          obtenido: L10          OK

RESULTADO: 6/6 pruebas superadas
TODO EN VERDE. Documenta tu auditoria en auditorias/auditoria_sesion23.md
```

* [x] Caso base y límites superados sin *Crash*.
* [x] No existen accesos fuera de límites ni *Memory Leaks*.
* [x] La complejidad teórica coincide con la implementada.

### Pruebas extra que hice yo
| Prueba | Esperado | Obtenido |
| :--- | :--- | :--- |
| 3 llamadas seguidas desde 0 | `L0 L1 L2`, contador 3 | `L0 L1 L2`, contador 3 |
| Generar una etiqueta y usarla en GOTO y en marca | `GOTO L0` y `L0:` | `GOTO L0` y `L0:` |
| `emitir_flujo("L2", 2)` | `GOTO L2` | `GOTO L2` |
| `emitir_flujo("L2", -1)` | `GOTO L2` | `GOTO L2` |
| `GOTO L123` en buffer de 4 | se corta sin desbordar | `"GOT"` |
| etiqueta NULL con es_salto = 0 | `""` | `""` |
| contador NULL | `""` | `""` |

Compilado con `-fsanitize=address`, sin errores de memoria.

### Reporte Crítico
*¿Qué caso borde o problema omitió la IA y cómo lo solucionaste?*
> En la primera versión de la IA, `emitir_flujo` tenía `if (es_salto == 1)` para escribir el GOTO, y si no, escribía la marca. En el harness solo se usan `es_salto = 1` y `es_salto = 0`, así que salía 6/6. Pero el contrato dice "distinto de cero", y si alguien llama con `es_salto = 2` o con el resultado de una comparación que no da exactamente 1, la IA escribía `L2:` en lugar de `GOTO L2`. O sea, en vez de saltar, ponía otra marca, y el programa seguiría de largo sin brincar.
>
> Lo detecté porque en la Fase 1 me pasó lo mismo y lo corregí. Hice la prueba con `es_salto = 2` y `-1`, y la versión de la IA me daba la marca.
>
> Lo arreglé poniendo en el contrato "DISTINTO DE CERO (1, 2, -1...)" con ejemplos. La segunda versión usa `es_salto ? ... : ...`, que toma cualquier valor no cero como verdadero.
>
> Conclusión: en C "verdadero" no es solo 1. Si comparas con `== 1` en vez de `!= 0`, te puede fallar con valores que el harness no prueba.

---
**Firma digital:** ReyesDiego002
