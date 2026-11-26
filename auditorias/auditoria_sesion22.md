# Bitácora de Requerimientos y Auditoría IA — Sesión 22
**Materia:** Lenguajes y Autómatas II (SCD-1016)  
**Tema:** Generación de TAC para Expresiones Aritmético-Lógicas  
**Docente Titular:** MCC. Iván Márquez Larios (`ijmarquezl`)  
**Estudiante:** Diego Reyes Martínez  
**Matrícula:** 24531376  
**Track:** C (el harness evalúa C; la versión en Rust es opcional y no se califica con este harness)  

---

## 1. Fase 1: El Artesano (Dificultad Manual)
*Reto:* Recorrer un AST en postorden y emitir una línea TAC por operador  
*Describe qué problema o restricción de memoria/sintaxis identificaste al codificar a mano:*
> Lo primero que me di cuenta fue que `generar_tac` sola no me alcanzaba, porque necesito que cada llamada me regrese el **nombre** donde quedó el resultado (una hoja como `a` o un temporal como `_t0`) para usarlo en la línea del papá. Hice una función aparte, `recorrer()`, que regresa ese nombre, y `generar_tac` solo la llama.
>
> Me atoré con el orden: si pido el temporal antes de bajar a los hijos, en `a + b * c` el `+` se queda con `_t0` y el `*` con `_t1`, y el harness espera al revés. Lo correcto es primero `izq`, luego `der`, y al final `nuevo_temporal` (postorden).
>
> También tuve que pasar muchos parámetros a `recorrer` (lineas, max, el contador de líneas, el pool y el contador de temporales). Se ve largo, pero así no uso variables globales.
>
> Lo último fue el `max`: antes de escribir reviso `if (*num < max)` para no salirme del arreglo `lineas`. Si ya no cabe, igual saco el temporal para que el papá tenga nombre, pero ya no escribo.

---

## 2. Fase 2: El Científico (Ingeniería de Requerimientos C-R-E-O)

### 2.1 Matriz de Especificación Formal
| Elemento C-R-E-O | Definición Técnica |
| :--- | :--- |
| **Contexto (C)** | C99 / GCC o Rust 2021 sin bibliotecas no estándar |
| **Rol (R)** | Ingeniero de software de sistemas y compiladores |
| **Especificación (E)** | Recorrido postorden del AST, una línea `_tX = a op b` por operador, límite de `max` líneas |
| **Output (O)** | Solo código modular y análisis de complejidad |

### 2.2 Prompt Estructurado
```text
CONTEXTO: Laboratorio de compiladores en C (C11, GCC): traducir un AST a código de
tres direcciones (TAC). Se compila con gcc -std=c11 -Wall -Wextra -Werror, sin
warnings. Solo librerías estándar y sin main() porque el harness ya tiene uno.
La práctica ya trae esto y hay que usarlo:

typedef struct { char nombre[64][16]; int usados; } Pool;
const char *nuevo_temporal(Pool *p, int *contador);

ROL: Actúa como ingeniero de software de sistemas y compiladores.

ESPECIFICACIÓN: Respeta exactamente esto de api_sesion22.h:

typedef struct Nodo {
    char tipo;      // 'H' hoja, 'B' binario
    char op;
    char valor[16];
    struct Nodo *izq;
    struct Nodo *der;
} Nodo;

int generar_tac(const Nodo *raiz, char lineas[][64], int max);

Reglas:
- Postorden: hijo izquierdo, hijo derecho y luego el operador.
- Las hojas no generan línea, solo regresan su valor.
- Cada operador pide temporal con nuevo_temporal y escribe "_tX = izq op der".
- Regresa cuántas líneas escribió.
- NUNCA más de max líneas ni más de 63 caracteres por línea (snprintf con 64).
- raiz NULL, lineas NULL o max <= 0 regresan 0.
- Incluye el mismo Pool y nuevo_temporal en el archivo.

Ejemplos: a + b * c -> "_t0 = b * c", "_t1 = a + _t0".
(a + b) * (c - d) -> línea 2 = "_t2 = _t0 * _t1".

OUTPUT: Solo el código de ia_sesion22_c.c, corto, y una línea con la complejidad.
```

---

## 3. Código Devuelto por el Modelo
```c
#include "api_sesion22.h"
#include <stdio.h>

/* Pool y nuevo_temporal iguales a los que da la practica */
typedef struct {
    char nombre[64][16];
    int usados;
} Pool;

const char *nuevo_temporal(Pool *p, int *contador) {
    if (p->usados >= 64) return "_t?";
    snprintf(p->nombre[p->usados], sizeof p->nombre[0], "_t%d", (*contador)++);
    return p->nombre[p->usados++];
}

typedef struct {
    char (*lineas)[64];
    int max;
    int n;        // lineas escritas
    int cont;     // contador de temporales
    Pool pool;
} Gen;

static const char *postorden(const Nodo *nd, Gen *g) {
    if (!nd) return "_";
    if (nd->tipo == 'H') return nd->valor;          // hoja: no emite

    const char *a = postorden(nd->izq, g);          // hijos primero
    const char *b = postorden(nd->der, g);
    const char *t = nuevo_temporal(&g->pool, &g->cont);

    if (g->n < g->max) {                            // nunca mas de max lineas
        snprintf(g->lineas[g->n], 64, "%s = %s %c %s", t, a, nd->op, b);
        g->n++;
    }
    return t;
}

int generar_tac(const Nodo *raiz, char lineas[][64], int max) {
    if (!raiz || !lineas || max <= 0) return 0;
    Gen g;
    g.lineas = lineas;
    g.max = max;
    g.n = 0;
    g.cont = 0;
    g.pool.usados = 0;
    postorden(raiz, &g);
    return g.n;
}

// Complejidad: O(n) en tiempo, n = nodos del arbol; memoria O(h) de recursion, h = altura
```

---

## 4. Fase 3: El Auditor (Resultados del Test Harness)
*Pruebas ejecutadas:* Precedencia de `*` sobre `+`, paréntesis, tres operadores, hoja sola y dos subárboles

### Salida del harness
```text
=== AUDITORIA SESION 22 - Traductor de AST a TAC con precedencia ===
Caso                               esperado:              obtenido:              resultado
[ 1] a + b * c                          esperado: _t0 = b * c  obtenido: _t0 = b * c  OK
[ 2] a + b * c (2a linea)
     esperado: _t1 = a + _t0
     obtenido: _t1 = a + _t0
     -> OK
[ 3] (a + b) * c
     esperado: _t1 = _t0 * c
     obtenido: _t1 = _t0 * c
     -> OK
[ 4] x * (y - z) + w
     esperado: _t2 = _t1 + w
     obtenido: _t2 = _t1 + w
     -> OK
[ 5] hoja sin operaciones               esperado: 0            obtenido: 0            OK
[ 6] (a + b) * (c - d)
     esperado: _t2 = _t0 * _t1
     obtenido: _t2 = _t0 * _t1
     -> OK

RESULTADO: 6/6 pruebas superadas
TODO EN VERDE. Documenta tu auditoria en auditorias/auditoria_sesion22.md
```

* [x] Caso base y límites superados sin *Crash*.
* [x] No existen accesos fuera de límites ni *Memory Leaks*.
* [x] La complejidad teórica coincide con la implementada.

### Pruebas extra que hice yo
| Prueba | Esperado | Obtenido |
| :--- | :--- | :--- |
| `(a + b) * (c - d)` completo | 3 líneas: `_t0 = a + b`, `_t1 = c - d`, `_t2 = _t0 * _t1` | igual |
| Mismo árbol con `max = 1` | 1 línea, sin tocar memoria de después | 1 línea, memoria intacta |
| `generar_tac(NULL, l, 8)` | 0 | 0 |
| `max = 0` | 0 | 0 |
| Operador con hijo derecho NULL | no truena | `_t0 = a + _` |

Compilado con `-fsanitize=address`, sin errores de memoria.

### Reporte Crítico
*¿Qué caso borde o problema omitió la IA y cómo lo solucionaste?*
> En la primera versión de la IA, el recorrido sí era postorden y los 6 casos salían OK, pero escribía la línea así: `snprintf(lineas[g->n++], 64, ...)`, **sin revisar `max`**. En el harness siempre le pasan `max = 8` o `max = 4` y los árboles tienen máximo 3 operadores, así que nunca se nota.
>
> El contrato dice "nunca escribas más de max líneas". Hice una prueba con el árbol `(a + b) * (c - d)` (3 operadores) y `max = 1`. La versión de la IA escribía las 3 líneas, o sea 2 fuera del arreglo, y `-fsanitize=address` marcó `stack-buffer-overflow`.
>
> Lo arreglé poniendo en el contrato, en mayúsculas, que nunca pase de `max` líneas. La segunda versión ya revisa `if (g->n < g->max)` antes de escribir. Igual saca el temporal aunque no escriba, para que el operador de arriba tenga un nombre que usar.
>
> Conclusión: el harness siempre da arreglos más grandes de lo necesario, así que nunca prueba qué pasa cuando no cabe todo. Esa prueba la tuve que hacer yo.

---
**Firma digital:** ReyesDiego002
