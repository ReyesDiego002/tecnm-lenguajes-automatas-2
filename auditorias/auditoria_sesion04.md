# Bitácora de Requerimientos y Auditoría IA — Sesión 04
**Materia:** Lenguajes y Autómatas II (SCD-1016)  
**Tema:** Anatomía de un Árbol Sintáctico Abstracto (AST)  
**Docente Titular:** MCC. Iván Márquez Larios (`ijmarquezl`)  
**Estudiante:** Diego Reyes Martínez
**Matrícula:** 24531376  
**Track:** [ C ]  

---

## 1. Fase 1: El Artesano (Dificultad Manual)
*Reto:* Nodos de AST en el heap y evaluación recursiva  
*Describe qué problema o restricción de memoria/sintaxis identificaste al codificar a mano:*
> Lo que más me costó fue entender la recursión de `evaluar`. Al principio quería aplicar el operador directo, pero me di cuenta de que primero hay que evaluar los dos hijos, porque un hijo puede ser otra operación y no un número. Por eso lo hice con un caso base (si el nodo es número, regreso su valor) y un caso recursivo (evalúo `izq` y `der`, y luego aplico `+`, `-` o `*`). También tuve que cuidar el orden de los hijos en `crear_binario`: en `10-4` si los cambio me da -6 en vez de 6. Otro detalle fue inicializar todos los campos del nodo, porque `malloc` no limpia la memoria y si dejaba `izq` y `der` sin valor en los nodos numéricos, quedaban con basura. Por último, revisé que `malloc` no regresara `NULL` y que `evaluar(NULL)` regresara -1 antes de tocar el nodo, porque si no marca fallo de segmentación.

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
Contexto: Estoy en la materia Lenguajes y Autómatas II. Trabajo en C (estándar C11) y compilo con gcc -std=c11 -Wall -Wextra -Werror, así que cualquier advertencia cuenta como error. Ya existe un archivo api_sesion04.h con esta estructura y estas firmas:
typedef enum { NODO_NUM, NODO_BINOP } TipoNodo;
typedef struct ASTNode { TipoNodo tipo; int valor; char op; struct ASTNode *izq; struct ASTNode *der; } ASTNode;
ASTNode *crear_numero(int v);
ASTNode *crear_binario(char op, ASTNode *izq, ASTNode *der);
int evaluar(ASTNode *n);

Rol: Actúa como ingeniero de software de sistemas y compiladores, con experiencia en árboles sintácticos abstractos y recursión en C.

Especificación:
- crear_numero y crear_binario reservan cada nodo con malloc y llenan todos los campos. crear_binario guarda los hijos en el orden recibido (izq y der).
- evaluar es recursiva: si el nodo es número devuelve su valor; si es operación, evalúa primero ambos hijos y después aplica el operador. Solo se soportan +, - y *.
- evaluar(NULL) devuelve -1 y no debe caerse.
- No uses main() ni cambies las firmas ni las estructuras.

Output: Dame solo el código del archivo ia_sesion04_c.c con comentarios breves, y al final un análisis corto de la complejidad en tiempo y en espacio.
```

---

## 3. Código Devuelto por el Modelo
```c
// Código devuelto por el LLM
#include "api_sesion04.h"
#include <stdio.h>
#include <stdlib.h>

ASTNode *crear_numero(int v) {
    ASTNode *n = malloc(sizeof(ASTNode));
    if (n == NULL) {
        return NULL;
    }
    n->tipo = NODO_NUM;
    n->valor = v;
    n->op = '\0';
    n->izq = NULL;
    n->der = NULL;
    return n;
}

ASTNode *crear_binario(char op, ASTNode *izq, ASTNode *der) {
    ASTNode *n = malloc(sizeof(ASTNode));
    if (n == NULL) {
        return NULL;
    }
    n->tipo = NODO_BINOP;
    n->valor = 0;
    n->op = op;
    n->izq = izq;
    n->der = der;
    return n;
}

int evaluar(ASTNode *n) {
    if (n == NULL) {
        return -1;
    }
    if (n->tipo == NODO_NUM) {
        return n->valor;
    }

    int a = evaluar(n->izq);
    int b = evaluar(n->der);

    switch (n->op) {
        case '+': return a + b;
        case '-': return a - b;
        case '*': return a * b;
        default:  return -1; /* operador no soportado */
    }
}

/* Complejidad: crear_numero y crear_binario son O(1) en tiempo y espacio.
   evaluar es O(n) en tiempo, porque visita cada nodo una vez, y O(h) en
   espacio por la pila de la recursion, con h la altura del arbol. */
```

---

## 4. Fase 3: El Auditor (Resultados del Test Harness)
*Pruebas ejecutadas:* Detección de errores de compilación GCC

* [x] Caso base y límites superados sin *Crash*.
* [x] No existen accesos fuera de límites ni *Memory Leaks*.
* [x] La complejidad teórica coincide con la implementada.

Salida del harness:
```text
[ 1] evaluar(nodo 42)                   esperado: 42           obtenido: 42           OK
[ 2] evaluar(3+5*2)                     esperado: 13           obtenido: 13           OK
[ 3] hijo izquierdo de (* 5 2)          esperado: 5            obtenido: 5            OK
[ 4] operador guardado                  esperado: *            obtenido: *            OK
[ 5] evaluar(NULL)                      esperado: -1           obtenido: -1           OK
[ 6] evaluar(10-4)                      esperado: 6            obtenido: 6            OK

RESULTADO: 6/6 pruebas superadas
```

### Reporte Crítico
*¿Qué caso borde o problema omitió la IA y cómo lo solucionaste?*
> En su primera respuesta la IA hacía el `switch` del operador sin un caso `default`, así que si el nodo traía un operador que no soportamos (por ejemplo `/`), la función llegaba al final sin regresar nada. El compilador lo marcaba como advertencia, y como compilamos con `-Werror` eso contaba como error. Si se ignoraba, el resultado era un valor basura. El harness no prueba ese caso, así que lo detecté al compilar y al revisar qué pasaba con un operador desconocido. Lo corregí agregando `default: return -1;`. También revisé que `crear_numero` y `crear_binario` inicializaran todos los campos, que revisaran si `malloc` devolvía `NULL`, y que los hijos quedaran en el orden `izq`/`der` correcto. Después compilé con `-Wall -Wextra -Werror` y el harness salió 6/6 en verde.

---
**Firma digital:** [ReyesDiego002]
