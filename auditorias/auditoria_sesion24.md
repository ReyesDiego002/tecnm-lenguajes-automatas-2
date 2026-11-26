# Bitácora de Requerimientos y Auditoría IA — Sesión 24
**Materia:** Lenguajes y Autómatas II (SCD-1016)  
**Tema:** Control de Flujo: Sentencias If e If-Else  
**Docente Titular:** MCC. Iván Márquez Larios (`ijmarquezl`)  
**Estudiante:** Diego Reyes Martínez  
**Matrícula:** 24531376  
**Track:** C (el harness evalúa C; la versión en Rust es opcional y no se califica con este harness)  

---

## 1. Fase 1: El Artesano (Dificultad Manual)
*Reto:* Saltos condicionales con cortocircuito para AND y OR  
*Describe qué problema o restricción de memoria/sintaxis identificaste al codificar a mano:*
> Al principio me confundí con cuál va con cuál. Lo pensé así: en `a && b`, si `a` es falso ya no tiene caso seguir, entonces salto a la salida. Por eso AND usa `IFFALSE`. En `a || b`, si `a` es verdadero ya se cumple todo, entonces salto. Por eso OR usa `IFTRUE`.
>
> Lo que me ayudó fue guardar la palabra (`"IFFALSE"` o `"IFTRUE"`) en una variable `salto` antes, para escribir el `snprintf` una sola vez con `%s` y no tener 4 `snprintf` distintos.
>
> Primero hice `if (op == '&') ... else ...`, o sea que todo lo que no fuera `&` lo tomaba como OR. Luego pensé que si llega un `'+'` por error iba a generar saltos que no tienen sentido, así que puse `else if (op == '|')` y un `else return 0;`.
>
> Para el segundo operando reviso `b != NULL` y también `n < max`, por si me pasan un arreglo de una sola línea.

---

## 2. Fase 2: El Científico (Ingeniería de Requerimientos C-R-E-O)

### 2.1 Matriz de Especificación Formal
| Elemento C-R-E-O | Definición Técnica |
| :--- | :--- |
| **Contexto (C)** | C99 / GCC o Rust 2021 sin bibliotecas no estándar |
| **Rol (R)** | Ingeniero de software de sistemas y compiladores |
| **Especificación (E)** | AND → `IFFALSE x GOTO L`, OR → `IFTRUE x GOTO L`, un salto por operando, operador inválido → 0 |
| **Output (O)** | Solo código modular y análisis de complejidad |

### 2.2 Prompt Estructurado
```text
CONTEXTO: Laboratorio de compiladores en C (C11, GCC) sobre traducir condiciones
de if a saltos con cortocircuito. Se compila con gcc -std=c11 -Wall -Wextra
-Werror, sin warnings. Solo librerías estándar y sin main() porque el harness ya
tiene uno.

ROL: Actúa como ingeniero de software de sistemas y compiladores.

ESPECIFICACIÓN: Respeta exactamente esta firma de api_sesion24.h:

int emitir_condicion(char lineas[][64], int max, const char *a, const char *b,
                     char op, const char *etiqueta);

Reglas:
- op '&' (AND): cada salto es "IFFALSE operando GOTO etiqueta".
- op '|' (OR): cada salto es "IFTRUE operando GOTO etiqueta".
- CUALQUIER OTRO op es inválido: regresa 0 y no escribe nada.
- Primero el salto de a y luego el de b, cada uno en su línea.
- Si b es NULL solo se escribe el de a.
- Regresa cuántas líneas escribió, nunca más de max.
- lineas, a o etiqueta NULL, o max <= 0, regresan 0.

Ejemplos: ("a","b",'&',"L9") -> "IFFALSE a GOTO L9", "IFFALSE b GOTO L9" (2).
("a",NULL,'&',"L0") -> "IFFALSE a GOTO L0" (1).

OUTPUT: Solo el código de ia_sesion24_c.c, corto, y una línea con la complejidad.
```

---

## 3. Código Devuelto por el Modelo
```c
#include "api_sesion24.h"
#include <stdio.h>
#include <stddef.h>

int emitir_condicion(char lineas[][64], int max, const char *a, const char *b, char op, const char *etiqueta) {
    if (!lineas || max <= 0 || !a || !etiqueta) return 0;

    const char *instr = NULL;
    if (op == '&') instr = "IFFALSE";   // AND: si es falso ya no sigue
    if (op == '|') instr = "IFTRUE";    // OR: si es verdadero ya no sigue
    if (!instr) return 0;               // operador invalido

    const char *ops[2] = { a, b };
    int n = 0;
    for (int i = 0; i < 2 && n < max; i++) {
        if (!ops[i]) break;             // b NULL: un solo salto
        snprintf(lineas[n], 64, "%s %s GOTO %s", instr, ops[i], etiqueta);
        n++;
    }
    return n;
}

// Complejidad: O(1) (maximo 2 lineas)
```

---

## 4. Fase 3: El Auditor (Resultados del Test Harness)
*Pruebas ejecutadas:* AND con dos operandos, OR con dos operandos, número de saltos y condición simple con b NULL

### Salida del harness
```text
=== AUDITORIA SESION 24 - Saltos condicionales con cortocircuito ===
Caso                               esperado:              obtenido:              resultado
[ 1] a && b (primer salto)
     esperado: IFFALSE a GOTO L9
     obtenido: IFFALSE a GOTO L9
     -> OK
[ 2] a && b (segundo salto)
     esperado: IFFALSE b GOTO L9
     obtenido: IFFALSE b GOTO L9
     -> OK
[ 3] a && b (dos saltos)                esperado: 2            obtenido: 2            OK
[ 4] a || b (primer salto)
     esperado: IFTRUE a GOTO L9
     obtenido: IFTRUE a GOTO L9
     -> OK
[ 5] a || b (segundo salto)
     esperado: IFTRUE b GOTO L9
     obtenido: IFTRUE b GOTO L9
     -> OK
[ 6] condicion simple (b NULL)
     esperado: IFFALSE a GOTO L0
     obtenido: IFFALSE a GOTO L0
     -> OK

RESULTADO: 6/6 pruebas superadas
TODO EN VERDE. Documenta tu auditoria en auditorias/auditoria_sesion24.md
```

* [x] Caso base y límites superados sin *Crash*.
* [x] No existen accesos fuera de límites ni *Memory Leaks*.
* [x] La complejidad teórica coincide con la implementada.

### Pruebas extra que hice yo
| Prueba | Esperado | Obtenido |
| :--- | :--- | :--- |
| `("a", NULL, '|', "L3")` | 1 línea: `IFTRUE a GOTO L3` | igual |
| operador `'x'` | 0 líneas | 0 |
| operador `'+'` | 0 líneas | 0 |
| AND con `max = 1` | solo 1 línea | 1 línea: `IFFALSE a GOTO L9` |
| `a` NULL | 0 | 0 |
| etiqueta NULL | 0 | 0 |
| `max = 0` | 0 | 0 |
| `("x>5", "y<3", '&', "L2")` | `IFFALSE x>5 GOTO L2`, `IFFALSE y<3 GOTO L2` | igual |

Compilado con `-fsanitize=address`, sin errores de memoria.

### Reporte Crítico
*¿Qué caso borde o problema omitió la IA y cómo lo solucionaste?*
> En la primera versión de la IA, el código tenía `const char *instr = (op == '&') ? "IFFALSE" : "IFTRUE";`. O sea, todo lo que no fuera `'&'` lo tomaba como OR. En el harness solo se usan `'&'` y `'|'`, así que salía 6/6. Pero con `op = 'x'` o `'+'` generaba `IFTRUE a GOTO L9` y `IFTRUE b GOTO L9` como si fuera un OR. Así, un error del analizador se convertiría en saltos que cambian lo que hace el programa sin que nadie se dé cuenta.
>
> Lo detecté porque en la Fase 1 me pasó lo mismo con el `else`. Hice la prueba con `'x'` y la versión de la IA me regresó 2 líneas con `IFTRUE`.
>
> Lo arreglé poniendo en el contrato que cualquier otro operador es inválido y regresa 0 sin escribir. La segunda versión empieza con `instr = NULL`, solo lo llena con `'&'` o `'|'`, y si sigue en NULL regresa 0.
>
> Conclusión: un `else` que agarra todo lo que sobra es peligroso. Es mejor revisar exactamente los casos válidos y rechazar lo demás.

---
**Firma digital:** ReyesDiego002
