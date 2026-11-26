# Bitácora de Requerimientos y Auditoría IA — Sesión 26
**Materia:** Lenguajes y Autómatas II (SCD-1016)  
**Tema:** TAC para Llamadas a Funciones y Retornos  
**Docente Titular:** MCC. Iván Márquez Larios (`ijmarquezl`)  
**Estudiante:** Diego Reyes Martínez  
**Matrícula:** 24531376  
**Track:** C (el harness evalúa C; la versión en Rust es opcional y no se califica con este harness)  

---

## 1. Fase 1: El Artesano (Dificultad Manual)
*Reto:* Emitir PARAM, CALL y RETURN para llamadas a funciones  
*Describe qué problema o restricción de memoria/sintaxis identificaste al codificar a mano:*
> Lo primero en que me atoré fue el número del CALL. Al principio puse `CALL f, 2` fijo, pero en `p()` el harness espera `CALL p, 0`. Entonces agregué una variable `params` que solo sube cuando de verdad escribo un PARAM, y esa es la que pongo en el CALL.
>
> También tuve que llevar dos contadores distintos: `params` (cuántos PARAM) y `escritas` (cuántas líneas metí en total, contando el CALL), porque la función regresa las líneas y el CALL necesita los parámetros.
>
> Revisé `a1` y `a2` con dos `if` separados en lugar de "si a1 es NULL ya no hay argumentos", porque el contrato dice "cada argumento no NULL genera un PARAM".
>
> La llamada anidada `f(g(x))` me espantó al principio, pero no tuve que hacer nada especial. El harness primero llama a `g` (que deja `_t0`) y luego a `f` con `_t0` de argumento, y como uso `*n` para seguir donde me quedé, no piso nada.
>
> Para el `RETURN` sin valor pensé en las funciones `void`: nada más escribo `RETURN`.

---

## 2. Fase 2: El Científico (Ingeniería de Requerimientos C-R-E-O)

### 2.1 Matriz de Especificación Formal
| Elemento C-R-E-O | Definición Técnica |
| :--- | :--- |
| **Contexto (C)** | C99 / GCC o Rust 2021 sin bibliotecas no estándar |
| **Rol (R)** | Ingeniero de software de sistemas y compiladores |
| **Especificación (E)** | Un PARAM por argumento no NULL (a1, a2 por separado), CALL con el conteo real, CALL sin asignar si resultado es NULL, RETURN con o sin valor |
| **Output (O)** | Solo código modular y análisis de complejidad |

### 2.2 Prompt Estructurado
```text
CONTEXTO: Laboratorio de compiladores en C (C11, GCC) sobre TAC para llamadas a
funciones. Se compila con gcc -std=c11 -Wall -Wextra -Werror, sin warnings. Solo
librerías estándar y sin main() porque el harness ya tiene uno. emitir_llamada
agrega líneas a lineas[] desde *n y sube *n.

ROL: Actúa como ingeniero de software de sistemas y compiladores.

ESPECIFICACIÓN: Respeta exactamente estas firmas de api_sesion26.h:

int emitir_llamada(char lineas[][64], int max, int *n, const char *funcion,
                   const char *a1, const char *a2, const char *resultado);
void emitir_return(char *dst, int cap, const char *valor);

Reglas:
- Revisa a1 y a2 POR SEPARADO: cada uno que no sea NULL genera "PARAM arg",
  en orden a1, a2. Si a1 es NULL pero a2 no, igual escribe "PARAM a2".
- Luego el CALL con la cantidad REAL de PARAM emitidos:
  "resultado = CALL funcion, k", o "CALL funcion, k" si resultado es NULL.
- Regresa cuántas líneas agregó. Nunca más de max.
  lineas, n o funcion NULL regresan 0.
- emitir_return: "RETURN valor", o solo "RETURN" si valor es NULL. Usa snprintf.

Ejemplos: f(x,y)->_t0: "PARAM x","PARAM y","_t0 = CALL f, 2".
p()->_t0: "_t0 = CALL p, 0".

OUTPUT: Solo el código de ia_sesion26_c.c, corto, y una línea con la complejidad.
```

---

## 3. Código Devuelto por el Modelo
```c
#include "api_sesion26.h"
#include <stdio.h>
#include <stddef.h>

int emitir_llamada(char lineas[][64], int max, int *n, const char *funcion, const char *a1, const char *a2, const char *resultado) {
    if (!lineas || !n || !funcion) return 0;

    const char *args[2] = { a1, a2 };
    int antes = *n, params = 0;

    // cada argumento no NULL es un PARAM (aunque a1 sea NULL, a2 se revisa)
    for (int i = 0; i < 2; i++) {
        if (args[i] && *n < max) {
            snprintf(lineas[(*n)++], 64, "PARAM %s", args[i]);
            params++;
        }
    }

    if (*n < max) {
        if (resultado) snprintf(lineas[(*n)++], 64, "%s = CALL %s, %d", resultado, funcion, params);
        else           snprintf(lineas[(*n)++], 64, "CALL %s, %d", funcion, params);
    }
    return *n - antes;   // lineas agregadas
}

void emitir_return(char *dst, int cap, const char *valor) {
    if (!dst || cap <= 0) return;
    if (valor) snprintf(dst, (size_t)cap, "RETURN %s", valor);
    else       snprintf(dst, (size_t)cap, "RETURN");
}

// Complejidad: O(1) (maximo 3 lineas por llamada)
```

---

## 4. Fase 3: El Auditor (Resultados del Test Harness)
*Pruebas ejecutadas:* PARAM en orden, CALL con dos argumentos, llamada anidada, CALL sin argumentos y RETURN con valor

### Salida del harness
```text
=== AUDITORIA SESION 26 - Llamadas, parametros y retorno ===
Caso                               esperado:              obtenido:              resultado
[ 1] PARAM x                            esperado: PARAM x      obtenido: PARAM x      OK
[ 2] PARAM y                            esperado: PARAM y      obtenido: PARAM y      OK
[ 3] CALL con dos argumentos
     esperado: _t0 = CALL f, 2
     obtenido: _t0 = CALL f, 2
     -> OK
[ 4] llamada anidada f(g(x))
     esperado: _t1 = CALL f, 1
     obtenido: _t1 = CALL f, 1
     -> OK
[ 5] CALL sin argumentos
     esperado: _t0 = CALL p, 0
     obtenido: _t0 = CALL p, 0
     -> OK
[ 6] RETURN con valor                   esperado: RETURN _t0   obtenido: RETURN _t0   OK

RESULTADO: 6/6 pruebas superadas
TODO EN VERDE. Documenta tu auditoria en auditorias/auditoria_sesion26.md
```

* [x] Caso base y límites superados sin *Crash*.
* [x] No existen accesos fuera de límites ni *Memory Leaks*.
* [x] La complejidad teórica coincide con la implementada.

### Pruebas extra que hice yo
**Llamada anidada `f(g(x))` completa:**
```text
PARAM x
_t0 = CALL g, 1
PARAM _t0
_t1 = CALL f, 1
```

| Prueba | Esperado | Obtenido |
| :--- | :--- | :--- |
| `imprimir(x)` con resultado NULL | `PARAM x`, `CALL imprimir, 1` | igual |
| `h(NULL, "y")` → `_t0` | `PARAM y`, `_t0 = CALL h, 1` | igual |
| `f(x, y)` con `max = 2` | solo los 2 PARAM, sin desbordar | `PARAM x`, `PARAM y` |
| `emitir_return(NULL)` | `RETURN` | `RETURN` |
| `RETURN _t0` en buffer de 4 | se corta sin desbordar | `RET` |
| función NULL | 0 líneas | 0 |

Compilado con `-fsanitize=address`, sin errores de memoria.

### Reporte Crítico
*¿Qué caso borde o problema omitió la IA y cómo lo solucionaste?*
> En la primera versión de la IA, el ciclo de los argumentos tenía `if (!args[i]) break;`, o sea que en cuanto encontraba un NULL dejaba de revisar. En el harness los NULL siempre vienen en `a2` (`g(x, NULL)`) o en los dos (`p(NULL, NULL)`), así que el `break` no se nota y salía 6/6.
>
> Pero el contrato dice que **cada** argumento no NULL genera un PARAM. Con `h(NULL, "y")`, la versión de la IA no escribía ningún PARAM y ponía `_t0 = CALL h, 0`. O sea, la función se llamaría sin su argumento `y`, y el número del CALL tampoco sería el real.
>
> Lo detecté porque en la Fase 1 hice dos `if` separados justo por eso, y probé `h(NULL, "y")` en las dos versiones.
>
> Lo arreglé poniendo en el contrato "revisa a1 y a2 POR SEPARADO" con ese ejemplo. La segunda versión ya usa `if (args[i] && ...)` en vez de `break`, y salen `PARAM y` y `CALL h, 1`.
>
> Conclusión: un `break` en el primer NULL supone que los NULL siempre van al final. El harness nunca prueba un NULL en medio.

---
**Firma digital:** ReyesDiego002
