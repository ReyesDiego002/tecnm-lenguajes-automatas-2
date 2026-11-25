# Bitácora de Requerimientos y Auditoría IA — Sesión 07
**Materia:** Lenguajes y Autómatas II (SCD-1016)  
**Tema:** Evaluación de Expresiones Aritméticas y Casos Límite  
**Docente Titular:** MCC. Iván Márquez Larios (`ijmarquezl`)  
**Estudiante:** Diego Reyes Martínez
**Matrícula:** 24531376  
**Track:** [ C ]  

---

## 1. Fase 1: El Artesano (Dificultad Manual)
*Reto:* Evaluador recursivo con detección de errores  
*Describe qué problema o restricción de memoria/sintaxis identificaste al codificar a mano:*
> Lo más importante fue revisar el divisor antes de dividir. En C, `7 / 0` con enteros hace que el programa termine con "Floating point exception", así que no basta con revisar el resultado después: hay que preguntar `if (derecha == 0)` antes de la operación. Lo mismo pasa con el módulo `%`, que también truena con 0 y que al principio se me olvidó. Otro problema fue la propagación del error: después de evaluar cada hijo reviso `*error != 0` y regreso de inmediato, así no se calcula el padre con valores basura. También me di cuenta de que no puedo poner `*error = 0` al inicio de la función, porque como es recursiva borraría el error que dejó un hijo, así que dejo que quien llama lo inicie en 0. Por último, el árbol NULL, un tipo de nodo inválido y un operador desconocido marcan error 2, y revisé que `evaluar(NULL, &e)` no se cayera.

---

## 2. Fase 2: El Científico (Ingeniería de Requerimientos C-R-E-O)

### 2.1 Matriz de Especificación Formal
| Elemento C-R-E-O | Definición Técnica |
| :--- | :--- |
| **Contexto (C)** | C99 / GCC o Rust 2021 sin bibliotecas no estándar |
| **Rol (R)** | Ingeniero de software de sistemas y compiladores |
| **Especificación (E)** | Salida estándar y códigos de retorno ABI |
| **Output (O)** | Solo código modular y análisis de complejidad |

### 2.2 Prompt Estructurado
```text
Contexto: Estoy en la materia Lenguajes y Autómatas II. Trabajo en C (estándar C11) y compilo con gcc -std=c11 -Wall -Wextra -Werror, así que cualquier advertencia cuenta como error. Ya existe un archivo api_sesion07.h con esta estructura y esta firma:
typedef enum { NODO_NUM, NODO_BINOP } TipoNodo;
typedef struct ASTNode { TipoNodo tipo; int valor; char op; struct ASTNode *izq; struct ASTNode *der; } ASTNode;
int evaluar(ASTNode *n, int *error);

Rol: Actúa como ingeniero de software de sistemas y compiladores, con experiencia en evaluación de expresiones y manejo de errores en C.

Especificación:
- evaluar calcula el árbol de forma recursiva y soporta +, -, *, / y %.
- Si hay división o módulo entre cero, pon *error = 1. Comprueba el divisor ANTES de operar, porque dividir entre cero en C termina el programa.
- Si el nodo es NULL, tiene un tipo inválido o un operador desconocido, pon *error = 2.
- El error se propaga hacia arriba: si un hijo falla, no se evalúa el padre ni el otro hijo.
- evaluar(NULL, &e) no debe caerse. Si error es NULL, no te caigas.
- Tú no reinicias *error a 0: el que llama lo inicia en 0.
- No uses main() ni cambies la firma ni las estructuras.

Output: Dame solo el código del archivo ia_sesion07_c.c con comentarios breves, y al final un análisis corto de la complejidad en tiempo y en espacio.
```

---

## 3. Código Devuelto por el Modelo
```c
// Código devuelto por el LLM
#include "api_sesion07.h"
#include <stdio.h>

int evaluar(ASTNode *n, int *error) {
    int basura = 0;
    if (error == NULL) {
        error = &basura; /* si no me dan puntero, uso una variable local */
    }

    if (n == NULL) {
        *error = 2;
        return 0;
    }
    if (n->tipo == NODO_NUM) {
        return n->valor;
    }
    if (n->tipo != NODO_BINOP) {
        *error = 2;
        return 0;
    }

    int a = evaluar(n->izq, error);
    if (*error != 0) {
        return 0;
    }
    int b = evaluar(n->der, error);
    if (*error != 0) {
        return 0;
    }

    switch (n->op) {
        case '+': return a + b;
        case '-': return a - b;
        case '*': return a * b;
        case '/':
            if (b == 0) {
                *error = 1;
                return 0;
            }
            return a / b;
        case '%':
            if (b == 0) {
                *error = 1;
                return 0;
            }
            return a % b;
        default:
            *error = 2; /* operador desconocido */
            return 0;
    }
}

/* Complejidad: tiempo O(n), porque se visita cada nodo una sola vez.
   Espacio O(h) por la pila de la recursion, con h la altura del arbol. */
```

---

## 4. Fase 3: El Auditor (Resultados del Test Harness)
*Pruebas ejecutadas:* Detección de errores de compilación GCC

* [x] Caso base y límites superados sin *Crash*.
* [x] No existen accesos fuera de límites ni *Memory Leaks*.
* [x] La complejidad teórica coincide con la implementada.

Salida del harness:
```text
[ 1] (2+3)*4                            esperado: 20           obtenido: 20           OK
[ 2] 7/0 -> error 1                     esperado: 1            obtenido: 1            OK
[ 3] 7%0 -> error 1                     esperado: 1            obtenido: 1            OK
[ 4] arbol NULL -> error 2              esperado: 2            obtenido: 2            OK
[ 5] 10-2-3                             esperado: 5            obtenido: 5            OK
[ 6] (8/2)+1                            esperado: 5            obtenido: 5            OK

RESULTADO: 6/6 pruebas superadas
```

### Reporte Crítico
*¿Qué caso borde o problema omitió la IA y cómo lo solucionaste?*
> En su primera respuesta la IA revisaba que el divisor no fuera 0 solo en la división `/`, pero en el módulo `%` hacía la operación directo. Con `7 % 0` el programa terminaba con "Floating point exception" y el harness se cortaba en el caso 3 sin imprimir el resto. Lo detecté justo por eso, porque la ejecución se detuvo en ese caso, y confirmé que el `%` también necesita la revisión del divisor. Lo corregí agregando el `if (b == 0)` con `*error = 1` en los dos casos. Otro problema fue que evaluaba el hijo derecho aunque el izquierdo ya hubiera marcado error, lo que gastaba trabajo y podía pisar el código de error. Lo arreglé revisando `*error != 0` después de cada hijo y regresando de inmediato. También verifiqué que el árbol NULL y el operador desconocido marcaran error 2. Después compilé con `-Wall -Wextra -Werror` y el harness salió 6/6 en verde.

---
**Firma digital:** [ReyesDiego002]
