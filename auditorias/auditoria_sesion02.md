# Bitácora de Requerimientos y Auditoría IA — Sesión 02
**Materia:** Lenguajes y Autómatas II (SCD-1016)  
**Tema:** Gestión en GitHub Web y Nodos Dinámicos en Heap  
**Docente Titular:** MCC. Iván Márquez Larios (`ijmarquezl`)  
**Estudiante:** Diego Reyes Martínez
**Matrícula:** 24531376  
**Track:** [ C ]  

---

## 1. Fase 1: El Artesano (Dificultad Manual)
*Reto:* Lista enlazada con memoria dinámica (malloc y free)  
*Describe qué problema o restricción de memoria/sintaxis identificaste al codificar a mano:*
> Lo más difícil fue liberar la lista sin perder el camino. Al principio quise hacer `free(actual)` y luego `actual = actual->siguiente`, pero eso lee memoria que ya se liberó. Lo arreglé guardando `siguiente` en una variable antes de hacer el `free`. Otro problema fue el campo `lexema`, que solo mide 16 bytes: si el texto es más largo y uso `strcpy`, se desborda el arreglo. Por eso usé `strncpy` con 15 caracteres y puse el `'\0'` al final a mano. También tuve que cuidar el caso de la lista vacía, porque ahí el nodo nuevo se convierte en la cabeza y hay que devolverlo en vez de la cabeza anterior. Por último, revisé que `malloc` no regresara `NULL` antes de usar el nodo, y que el `siguiente` del último nodo quedara en `NULL`, porque si no al recorrer la lista el programa se sale y truena.

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
Contexto: Estoy en la materia Lenguajes y Autómatas II. Trabajo en C (estándar C11) y compilo con gcc -std=c11 -Wall -Wextra -Werror, así que cualquier advertencia cuenta como error. Ya existe un archivo api_sesion02.h con esta estructura y estas firmas:
typedef struct NodoToken { char lexema[16]; struct NodoToken *siguiente; } NodoToken;
NodoToken *agregar_token(NodoToken *cabeza, const char *lexema);
int contar_tokens(NodoToken *cabeza);
void liberar_lista(NodoToken *cabeza);

Rol: Actúa como ingeniero de software de sistemas y compiladores, con experiencia en memoria dinámica y listas enlazadas en C.

Especificación:
- agregar_token reserva un nodo con malloc, copia el lexema sin desbordar el arreglo de 16 bytes y lo agrega al final. Devuelve la cabeza de la lista (si estaba vacía, devuelve el nodo nuevo).
- contar_tokens devuelve cuántos nodos tiene la lista, y 0 si la cabeza es NULL.
- liberar_lista libera todos los nodos sin usar memoria ya liberada, y no debe caerse con NULL.
- El campo siguiente del último nodo debe valer NULL.
- No uses main() ni cambies las firmas ni la estructura.

Output: Dame solo el código del archivo ia_sesion02_c.c con comentarios breves, y al final un análisis corto de la complejidad en tiempo y en espacio.
```

---

## 3. Código Devuelto por el Modelo
```c
// Código devuelto por el LLM
#include "api_sesion02.h"
#include <stdio.h>
#include <stdlib.h>

NodoToken *agregar_token(NodoToken *cabeza, const char *lexema) {
    if (lexema == NULL) {
        return cabeza;
    }

    NodoToken *nuevo = malloc(sizeof(NodoToken));
    if (nuevo == NULL) {
        return cabeza;
    }

    /* snprintf corta el texto a 15 caracteres y siempre pone el '\0' */
    snprintf(nuevo->lexema, sizeof(nuevo->lexema), "%s", lexema);
    nuevo->siguiente = NULL;

    if (cabeza == NULL) {
        return nuevo;
    }

    NodoToken *ultimo = cabeza;
    while (ultimo->siguiente != NULL) {
        ultimo = ultimo->siguiente;
    }
    ultimo->siguiente = nuevo;
    return cabeza;
}

int contar_tokens(NodoToken *cabeza) {
    int total = 0;
    while (cabeza != NULL) {
        total++;
        cabeza = cabeza->siguiente;
    }
    return total;
}

void liberar_lista(NodoToken *cabeza) {
    while (cabeza != NULL) {
        NodoToken *borrar = cabeza;
        cabeza = cabeza->siguiente; /* avanzo antes de liberar el nodo */
        free(borrar);
    }
}

/* Complejidad: agregar_token es O(n) porque recorre la lista hasta el final.
   contar_tokens y liberar_lista son O(n). El espacio extra es O(1) en las
   tres funciones, aparte del nodo que se reserva en agregar_token. */
```

---

## 4. Fase 3: El Auditor (Resultados del Test Harness)
*Pruebas ejecutadas:* Detección de errores de compilación GCC

* [x] Caso base y límites superados sin *Crash*.
* [x] No existen accesos fuera de límites ni *Memory Leaks*.
* [x] La complejidad teórica coincide con la implementada.

Salida del harness:
```text
[ 1] agregar_token(NULL,"if")           esperado: if           obtenido: if           OK
[ 2] contar_tokens(3 tokens)            esperado: 3            obtenido: 3            OK
[ 3] orden: segundo token               esperado: dos          obtenido: dos          OK
[ 4] ultimo->siguiente == NULL          esperado: 1            obtenido: 1            OK
[ 5] contar_tokens(NULL)                esperado: 0            obtenido: 0            OK
[ 6] liberar_lista(NULL)                esperado: sin crash    obtenido: sin crash    OK

RESULTADO: 6/6 pruebas superadas
```

### Reporte Crítico
*¿Qué caso borde o problema omitió la IA y cómo lo solucionaste?*
> En su primera respuesta la IA copiaba el lexema con `strcpy`, sin revisar el tamaño. Como el arreglo `lexema` solo tiene 16 bytes, un token de más de 15 caracteres (por ejemplo "identificador_largo") se desbordaba y escribía fuera del nodo, y el harness no lo prueba, así que pasaba los 6 casos aun con ese error. Lo detecté revisando el código línea por línea y comparando el tamaño del arreglo con lo que se copiaba. Lo corregí usando `snprintf` con `sizeof(nuevo->lexema)`, que corta el texto y siempre termina la cadena con `'\0'`. También verifiqué que `liberar_lista` guardara el siguiente nodo antes de hacer `free`, y que revisara si `malloc` devolvía `NULL`. Después compilé con `-Wall -Wextra -Werror` y el harness salió 6/6 en verde.

---
**Firma digital:** [ReyesDiego002]
