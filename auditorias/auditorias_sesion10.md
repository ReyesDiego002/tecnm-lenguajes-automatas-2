# Bitácora de Requerimientos y Auditoría IA — Sesión 10
**Materia:** Lenguajes y Autómatas II (SCD-1016)  
**Tema:** Hito 1: Mini-Evaluador de Expresiones y AST Completo  
**Docente Titular:** MCC. Iván Márquez Larios (`ijmarquezl`)  
**Estudiante:** Diego Reyes Martínez  
**Matrícula:** 24531376  
**Track:** C (el harness evalúa C; la versión en Rust es opcional y no se califica con este harness)  

---

## 1. Fase 1: El Artesano (Dificultad Manual)
*Reto:* Analizador descendente recursivo con precedencia, paréntesis y errores  
*Describe qué problema o restricción de memoria/sintaxis identificaste al codificar a mano:*
> Lo más difícil fue entender que la precedencia sale del orden en que se llaman las funciones: `expr` llama a `termino` y `termino` llama a `primaria`. Por eso `*` y `/` se resuelven antes que `+` y `-` sin tener que preguntar cuál operador va primero.
>
> Me atoré con los paréntesis: en `primaria`, cuando veo `(` tengo que llamar otra vez a `expr` (recursión) y después revisar que venga `)`. Si no viene, es error 2.
>
> Otro problema fue que el error no se "perdiera". Si `primaria` marcaba error pero `termino` seguía calculando, a veces salía un resultado raro. Lo arreglé revisando `p->error` después de cada llamada y saliendo con `return 0`.
>
> Para la división entre cero, la revisión la puse justo antes de dividir (`der == 0`), no buscando el carácter `'0'`. Así también detecta casos como `10/(5-5)`.
>
> Con `-Wall -Wextra -Werror` tuve que usar `isdigit((unsigned char)*p->s)` para que no hubiera advertencias con `char`.

---

## 2. Fase 2: El Científico (Ingeniería de Requerimientos C-R-E-O)

### 2.1 Matriz de Especificación Formal
| Elemento C-R-E-O | Definición Técnica |
| :--- | :--- |
| **Contexto (C)** | C99 / GCC o Rust 2021 sin bibliotecas no estándar |
| **Rol (R)** | Ingeniero de software de sistemas y compiladores |
| **Especificación (E)** | Analizador descendente recursivo: precedencia, paréntesis, error 1 (div/0) y error 2 (sintaxis) |
| **Output (O)** | Solo código modular y análisis de complejidad |

### 2.2 Prompt Estructurado
```text
CONTEXTO: Estoy en un laboratorio de compiladores en C (C11, GCC) sin bibliotecas
no estándar. Se compila con: gcc -std=c11 -Wall -Wextra -Werror, así que no puede
tener advertencias. Mi archivo se compila junto con un harness que ya trae main(),
por eso NO debe tener main().

ROL: Actúa como ingeniero de software de sistemas y compiladores que sabe escribir
analizadores descendentes recursivos.

ESPECIFICACIÓN: Implementa esta función respetando EXACTAMENTE la firma de
api_sesion10.h:

long evaluar_expresion(const char *fuente, int *error);

Usa esta gramática con las funciones primaria, termino y expr:
  expr     -> termino (('+' | '-') termino)*
  termino  -> primaria (('*' | '/') primaria)*
  primaria -> NUMERO | '(' expr ')'

Reglas:
- Solo enteros y + - * /. Sin variables ni menos unario.
- Respeta precedencia y paréntesis; ignora espacios.
- Asociatividad a la izquierda: "10-2-3" = 5.
- *error = 1 si hay división entre cero (incluye "10/(5-5)").
- *error = 2 si la sintaxis es inválida (falta número, falta ')',
  sobra texto o cadena vacía).
- Si hay error devuelve 0. Si error es NULL no debe tronar.
- Ejemplos: "2+3*4"=14, "(2+3)*4"=20, "5-10"=-5, "  7 "=7.

OUTPUT: Solo el código completo del archivo ia_sesion10_c.c y al final una línea
con la complejidad.
```

---

## 3. Código Devuelto por el Modelo
```c
#include "api_sesion10.h"
#include <stddef.h>
#include <ctype.h>

/* Gramatica:
 *   expr     -> termino (('+' | '-') termino)*
 *   termino  -> primaria (('*' | '/') primaria)*
 *   primaria -> NUMERO | '(' expr ')'
 */
typedef struct { const char *s; int error; } Parser;

static void saltar(Parser *p) {
    while (*p->s == ' ') p->s++;
}

static long expr(Parser *p);

static long primaria(Parser *p) {
    long v = 0;
    saltar(p);
    if (*p->s == '(') {
        p->s++;
        v = expr(p);
        if (p->error) return 0;
        saltar(p);
        if (*p->s != ')') { p->error = 2; return 0; }
        p->s++;
        return v;
    }
    if (!isdigit((unsigned char)*p->s)) { p->error = 2; return 0; }
    while (isdigit((unsigned char)*p->s)) {
        v = v * 10 + (*p->s - '0');
        p->s++;
    }
    return v;
}

static long termino(Parser *p) {
    long v = primaria(p);
    while (!p->error) {
        saltar(p);
        char op = *p->s;
        if (op != '*' && op != '/') break;
        p->s++;
        long d = primaria(p);
        if (p->error) return 0;
        if (op == '/' && d == 0) { p->error = 1; return 0; }
        v = (op == '*') ? v * d : v / d;
    }
    return v;
}

static long expr(Parser *p) {
    long v = termino(p);
    while (!p->error) {
        saltar(p);
        char op = *p->s;
        if (op != '+' && op != '-') break;
        p->s++;
        long d = termino(p);
        if (p->error) return 0;
        v = (op == '+') ? v + d : v - d;
    }
    return v;
}

long evaluar_expresion(const char *fuente, int *error) {
    static int sin_error;
    if (error == NULL) error = &sin_error;
    *error = 0;
    if (fuente == NULL) { *error = 2; return 0; }
    Parser p = { fuente, 0 };
    long v = expr(&p);
    saltar(&p);
    if (p.error == 0 && *p.s != '\0') p.error = 2;   /* basura al final */
    *error = p.error;
    return p.error != 0 ? 0 : v;
}

/* Complejidad: O(n) en tiempo, n = largo de la cadena. Memoria O(k), k = profundidad de parentesis (recursion). */
```

---

## 4. Fase 3: El Auditor (Resultados del Test Harness)
*Pruebas ejecutadas:* Precedencia, paréntesis, espacios, resultado negativo, división entre cero y sintaxis inválida

### Salida del harness
```text
=== AUDITORIA SESION 10 - Mini-evaluador de expresiones (Hito 1) ===
Caso                               esperado:              obtenido:              resultado
[ 1] "2+3*4"                            esperado: 14           obtenido: 14           OK
[ 2] "(2+3)*4"                          esperado: 20           obtenido: 20           OK
[ 3] "10/0" -> error 1                  esperado: 1            obtenido: 1            OK
[ 4] "2+" -> error 2                    esperado: 2            obtenido: 2            OK
[ 5] "  7 "                             esperado: 7            obtenido: 7            OK
[ 6] "5-10"                             esperado: -5           obtenido: -5           OK

RESULTADO: 6/6 pruebas superadas
TODO EN VERDE. Documenta tu auditoria en auditorias/auditoria_sesion10.md
```

* [x] Caso base y límites superados sin *Crash*.
* [x] No existen accesos fuera de límites ni *Memory Leaks*.
* [x] La complejidad teórica coincide con la implementada.

### Pruebas extra (hechas a mano, no las mide el harness)
| Expresión | Esperado | Obtenido |
| :--- | :--- | :--- |
| `10-2-3` | 5 | 5 |
| `8/2/2` | 2 | 2 |
| `10/(5-5)` | error 1 | error 1 |
| `(1+2` | error 2 | error 2 |
| `1+2)` | error 2 | error 2 |
| `2 3` | error 2 | error 2 |
| `""` (vacía) | error 2 | error 2 |
| `2*(3+4)-1` | 13 | 13 |

### Reporte Crítico
*¿Qué caso borde o problema omitió la IA y cómo lo solucionaste?*
> La primera propuesta de la IA escribía la gramática con recursión por la derecha: `expr -> termino '-' expr`. Así, `10-2-3` se calculaba como `10-(2-3) = 11` en lugar de `(10-2)-3 = 5`, y `8/2/2` daba 8 en lugar de 2. El harness no lo detecta, porque sus casos solo tienen un operador de cada nivel (`5-10`, `2+3*4`) y con eso los 6 casos salían OK.
>
> Lo detecté haciendo pruebas extra a mano con dos restas y dos divisiones seguidas, y comparando contra mi versión del Artesano, que usa un ciclo `while`.
>
> Lo solucioné pidiéndole explícitamente la asociatividad a la izquierda y la gramática con `( ... )*`, que se programa con un `while` que va acumulando el resultado. También revisé que la división entre cero se detectara con el valor ya calculado (`d == 0`) y no buscando el carácter `'0'`, para que `10/(5-5)` diera error 1.
>
> Conclusión: tener 6/6 en el harness no prueba que el evaluador esté bien. La asociatividad es un error que solo aparece si pruebas expresiones con varios operadores del mismo nivel.

---
**Firma digital:** ReyesDiego002
