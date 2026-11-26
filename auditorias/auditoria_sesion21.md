# Bitácora de Requerimientos y Auditoría IA — Sesión 21
**Materia:** Lenguajes y Autómatas II (SCD-1016)  
**Tema:** Fundamentos de Código de Tres Direcciones (TAC)  
**Docente Titular:** MCC. Iván Márquez Larios (`ijmarquezl`)  
**Estudiante:** Diego Reyes Martínez  
**Matrícula:** 24531376  
**Track:** C (el harness evalúa C; la versión en Rust es opcional y no se califica con este harness)  

---

## 1. Fase 1: El Artesano (Dificultad Manual)
*Reto:* Generar temporales únicos y formatear cuádruplos en un buffer  
*Describe qué problema o restricción de memoria/sintaxis identificaste al codificar a mano:*
> Esta se parece a la sesión 09 (el formateador de errores), porque otra vez es escribir en un buffer con `snprintf` respetando `cap`. Eso ya lo tenía claro.
>
> Me atoré con el contador. Primero escribí `snprintf(dst, cap, "_t%d", ++(*contador));` y me dio `_t1` en vez de `_t0`, porque `++` adelante incrementa antes de usar. Lo cambié a usarlo primero y luego hacer `(*contador)++` en otra línea, que se entiende mejor.
>
> También me confundí con los paréntesis: `*contador++` no es lo mismo que `(*contador)++`. El primero mueve el puntero, no el número. Por eso le puse los paréntesis.
>
> Para los NULL lo hice fácil: si viene NULL le pongo `"_"` a esa variable antes del `snprintf`. Así el formato siempre tiene sus 4 campos.

---

## 2. Fase 2: El Científico (Ingeniería de Requerimientos C-R-E-O)

### 2.1 Matriz de Especificación Formal
| Elemento C-R-E-O | Definición Técnica |
| :--- | :--- |
| **Contexto (C)** | C99 / GCC o Rust 2021 sin bibliotecas no estándar |
| **Rol (R)** | Ingeniero de software de sistemas y compiladores |
| **Especificación (E)** | Temporal `_tK` con post-incremento del contador y cuádruplo `(op, a1, a2, res)` con `_` para NULL |
| **Output (O)** | Solo código modular y análisis de complejidad |

### 2.2 Prompt Estructurado
```text
CONTEXTO: Laboratorio de compiladores en C (C11, GCC) sobre código de tres
direcciones y cuádruplos. Se compila con gcc -std=c11 -Wall -Wextra -Werror, sin
warnings. Solo librerías estándar, sin malloc, sin globales y sin main() porque
el harness ya tiene uno. Las funciones escriben en un buffer, no imprimen.

ROL: Actúa como ingeniero de software de sistemas y compiladores.

ESPECIFICACIÓN: Respeta exactamente estas firmas de api_sesion21.h:

void nuevo_temporal(char *dst, int cap, int *contador);
void emitir_cuadruplo(char *dst, int cap, const char *op, const char *a1,
                      const char *a2, const char *res);

Reglas:
- nuevo_temporal escribe "_t" + valor actual del contador y DESPUÉS lo
  incrementa. Contador 0 -> "_t0" y el contador queda en 1.
- emitir_cuadruplo escribe "(op, a1, a2, res)" con coma y espacio. Cualquier
  argumento NULL se escribe como "_".
- Usa snprintf con cap: nunca más de cap-1 caracteres y siempre '\0'.
  Si dst es NULL o cap <= 0 no hagas nada.

Ejemplos: "(+, a, b, _t0)", "(=, a, _, x)", "(goto, L1, _, _)".

OUTPUT: Solo el código de ia_sesion21_c.c, corto, y una línea con la complejidad.
```

---

## 3. Código Devuelto por el Modelo
```c
#include "api_sesion21.h"
#include <stdio.h>

// si viene NULL regresa "_"
static const char *o_guion(const char *s) {
    return s ? s : "_";
}

void nuevo_temporal(char *dst, int cap, int *contador) {
    if (!dst || cap <= 0) return;
    if (!contador) { dst[0] = '\0'; return; }
    snprintf(dst, (size_t)cap, "_t%d", *contador);
    *contador += 1;   // se usa y DESPUES se incrementa
}

void emitir_cuadruplo(char *dst, int cap, const char *op, const char *a1, const char *a2, const char *res) {
    if (!dst || cap <= 0) return;
    snprintf(dst, (size_t)cap, "(%s, %s, %s, %s)",
             o_guion(op), o_guion(a1), o_guion(a2), o_guion(res));
}

// Complejidad: O(L) en las dos, L = largo del texto (limitado por cap)
```

---

## 4. Fase 3: El Auditor (Resultados del Test Harness)
*Pruebas ejecutadas:* Temporal desde 0 y desde 7, cuádruplo binario, con temporal, asignación con NULL y goto con dos NULL

### Salida del harness
```text
=== AUDITORIA SESION 21 - Cuadruplos y temporales unicos ===
Caso                               esperado:              obtenido:              resultado
[ 1] nuevo_temporal(_t0)                esperado: _t0          obtenido: _t0          OK
[ 2] nuevo_temporal(_t7)                esperado: _t7          obtenido: _t7          OK
[ 3] cuadruplo de a + b
     esperado: (+, a, b, _t0)
     obtenido: (+, a, b, _t0)
     -> OK
[ 4] cuadruplo de _t0 * c
     esperado: (*, _t0, c, _t1)
     obtenido: (*, _t0, c, _t1)
     -> OK
[ 5] asignacion simple                  esperado: (=, a, _, x) obtenido: (=, a, _, x) OK
[ 6] goto sin operandos
     esperado: (goto, L1, _, _)
     obtenido: (goto, L1, _, _)
     -> OK

RESULTADO: 6/6 pruebas superadas
TODO EN VERDE. Documenta tu auditoria en auditorias/auditoria_sesion21.md
```

* [x] Caso base y límites superados sin *Crash*.
* [x] No existen accesos fuera de límites ni *Memory Leaks*.
* [x] La complejidad teórica coincide con la implementada.

### Pruebas extra que hice yo
| Prueba | Esperado | Obtenido |
| :--- | :--- | :--- |
| Contador después de `_t0` | 1 | 1 |
| Contador después de `_t7` | 8 | 8 |
| 11 llamadas seguidas desde 0 | `_t0` ... `_t10`, contador 11 | `_t0` ... `_t10`, contador 11 |
| Cuádruplo en buffer de 8 bytes | se corta sin desbordar | `"(+, a, "` |
| Temporal `_t10` en buffer de 3 bytes | se corta sin desbordar | `"_t"` |
| contador NULL | cadena vacía, sin tronar | `""` |

Compilado con `-fsanitize=address`, sin errores de memoria.

### Reporte Crítico
*¿Qué caso borde o problema omitió la IA y cómo lo solucionaste?*
> En la primera versión de la IA, `nuevo_temporal` hacía `snprintf(dst, cap, "_t%d", *contador);` y ya, **nunca incrementaba el contador**. El harness no lo detecta: en el caso 1 pone `c = 0` y espera `_t0`, en el caso 2 pone `c = 7` y espera `_t7`, pero nunca revisa cuánto vale `c` después. Por eso salía 6/6.
>
> El problema es grave: si generas el código de `a + b * c`, todos los temporales salen `_t0` y uno pisa al otro, que es justo lo que la pregunta guía dice que no debe pasar.
>
> Lo detecté porque el primer criterio de aceptación dice "y lo deja incrementado". Hice una prueba llamando 3 veces seguidas y me salió `_t0 _t0 _t0`.
>
> Lo arreglé poniendo en el contrato, bien claro, que después de escribir incremente el contador, con el ejemplo "contador 0 → `_t0` y queda en 1". La segunda versión ya trae `*contador += 1;`. Lo comprobé con 11 llamadas seguidas (de `_t0` a `_t10`).
>
> Conclusión: el harness revisa lo que escribe la función, pero no lo que deja cambiado. Hay que leer los criterios, no solo la tabla de casos.

---
**Firma digital:** ReyesDiego002
