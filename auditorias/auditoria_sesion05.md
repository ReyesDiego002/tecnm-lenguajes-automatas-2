# Bitácora de Requerimientos y Auditoría IA — Sesión 05
**Materia:** Lenguajes y Autómatas II (SCD-1016)  
**Tema:** Recorridos del AST y Notación Polaca Inversa  
**Docente Titular:** MCC. Iván Márquez Larios (`ijmarquezl`)  
**Estudiante:** Diego Reyes Martínez
**Matrícula:** 24531376  
**Track:** [ C ]  

---

## 1. Fase 1: El Artesano (Dificultad Manual)
*Reto:* Recorrido postorden y escritura en un buffer con capacidad limitada  
*Describe qué problema o restricción de memoria/sintaxis identificaste al codificar a mano:*
> Lo que más me costó fue escribir en el buffer sin pasarme de la capacidad. Como no se puede usar `printf`, tuve que armar una función auxiliar que agrega cada texto al final y que revisa antes si cabe, dejando lugar para el `'\0'`. Otro problema fue el espacio: al principio ponía un espacio después de cada token y la salida quedaba con espacio al final (`3 5 2 * + `), y el harness compara el texto exacto. Lo arreglé poniendo el espacio solo antes de un token cuando el buffer ya tiene algo. También tuve que convertir los números a texto con `snprintf`, y el operador a una cadena de un solo carácter. Para el árbol NULL, lo resolví iniciando el buffer con `buf[0] = '\0'` antes de recorrer, así queda la cadena vacía. Por último, el orden: en postorden primero van los dos hijos y al final la raíz, si lo hago al revés me sale la notación prefija en vez de la RPN.

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
Contexto: Estoy en la materia Lenguajes y Autómatas II. Trabajo en C (estándar C11) y compilo con gcc -std=c11 -Wall -Wextra -Werror, así que cualquier advertencia cuenta como error. Ya existe un archivo api_sesion05.h con esta estructura y esta firma:
typedef enum { NODO_NUM, NODO_BINOP } TipoNodo;
typedef struct ASTNode { TipoNodo tipo; int valor; char op; struct ASTNode *izq; struct ASTNode *der; } ASTNode;
void postorden(ASTNode *n, char *buf, int cap);

Rol: Actúa como ingeniero de software de sistemas y compiladores, con experiencia en recorridos de árboles y notación polaca inversa (RPN).

Especificación:
- postorden recorre el árbol en orden izquierdo, derecho, raíz y escribe los tokens en buf separados por un solo espacio, sin espacio al final. Los números se escriben como texto y los operadores como un carácter.
- No uses printf: escribe solo en el buffer.
- Respeta la capacidad: nunca escribas más de cap-1 caracteres más el '\0'. El buffer siempre debe terminar en '\0'.
- Un árbol NULL produce cadena vacía. Si buf es NULL o cap <= 0, no hagas nada.
- Puedes usar funciones auxiliares static, pero no uses main() ni cambies la firma.

Output: Dame solo el código del archivo ia_sesion05_c.c con comentarios breves, y al final un análisis corto de la complejidad en tiempo y en espacio.
```

---

## 3. Código Devuelto por el Modelo
```c
// Código devuelto por el LLM
#include "api_sesion05.h"
#include <stdio.h>
#include <string.h>

/* usado lleva la cuenta de cuantos caracteres ya tiene el buffer */
static void escribir(ASTNode *n, char *buf, int cap, int *usado) {
    if (n == NULL) {
        return;
    }

    escribir(n->izq, buf, cap, usado);
    escribir(n->der, buf, cap, usado);

    char texto[16];
    if (n->tipo == NODO_NUM) {
        snprintf(texto, sizeof(texto), "%d", n->valor);
    } else {
        texto[0] = n->op;
        texto[1] = '\0';
    }

    int espacio = (*usado > 0) ? 1 : 0; /* solo pongo espacio si ya hay algo */
    int largo = (int)strlen(texto);

    /* reviso la capacidad antes de escribir */
    if (*usado + espacio + largo > cap - 1) {
        return;
    }

    if (espacio) {
        buf[*usado] = ' ';
        (*usado)++;
    }
    memcpy(buf + *usado, texto, (size_t)largo);
    *usado += largo;
    buf[*usado] = '\0';
}

void postorden(ASTNode *n, char *buf, int cap) {
    if (buf == NULL || cap <= 0) {
        return;
    }

    buf[0] = '\0';
    int usado = 0;
    escribir(n, buf, cap, &usado);
}

/* Complejidad: tiempo O(n), porque se visita cada nodo una sola vez y cada
   escritura es constante. Espacio O(h) por la pila de la recursion, con h la
   altura del arbol, mas O(1) de variables auxiliares. */
```

---

## 4. Fase 3: El Auditor (Resultados del Test Harness)
*Pruebas ejecutadas:* Detección de errores de compilación GCC

* [x] Caso base y límites superados sin *Crash*.
* [x] No existen accesos fuera de límites ni *Memory Leaks*.
* [x] La complejidad teórica coincide con la implementada.

Salida del harness:
```text
[ 1] hoja 42                            esperado: 42           obtenido: 42           OK
[ 2] 3+(5*2)                            esperado: 3 5 2 * +    obtenido: 3 5 2 * +    OK
[ 3] arbol NULL                         esperado:              obtenido:              OK
[ 4] (1+2)*(3+4)
     esperado: 1 2 + 3 4 + *
     obtenido: 1 2 + 3 4 + *
     -> OK
[ 5] 8-3                                esperado: 8 3 -        obtenido: 8 3 -        OK

RESULTADO: 5/5 pruebas superadas
```

### Reporte Crítico
*¿Qué caso borde o problema omitió la IA y cómo lo solucionaste?*
> En su primera respuesta la IA usaba `strcat` y `sprintf` para ir armando el texto, sin revisar la capacidad `cap`. Con los buffers grandes del harness funcionaba, pero con un buffer chico el texto se escribía fuera del arreglo y se pasaba de memoria. El harness no prueba ese caso porque todos sus buffers tienen espacio de sobra, así que lo detecté revisando el código y preguntándome qué pasaba con `cap` pequeño. Lo corregí llevando la cuenta de los caracteres usados (`usado`) y verificando que `usado + espacio + largo` no pase de `cap - 1` antes de escribir cada token. Con esto el buffer siempre queda terminado en `'\0'`. También verifiqué que no quedara espacio al final y que `postorden(NULL, ...)` dejara la cadena vacía. Después compilé con `-Wall -Wextra -Werror` y el harness salió 5/5 en verde.

---
**Firma digital:** [ReyesDiego002]
