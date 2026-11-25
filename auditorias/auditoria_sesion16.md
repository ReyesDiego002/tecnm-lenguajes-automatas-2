# Bitácora de Requerimientos y Auditoría IA — Sesión 16
**Materia:** Lenguajes y Autómatas II (SCD-1016)  
**Tema:** Firmas de Función y Parámetros Formales  
**Docente Titular:** MCC. Iván Márquez Larios (`ijmarquezl`)  
**Estudiante:** Diego Reyes Martínez  
**Matrícula:** 24531376  
**Track:** C (el harness evalúa C; la versión en Rust es opcional y no se califica con este harness)  

---

## 1. Fase 1: El Artesano (Dificultad Manual)
*Reto:* Verificación de aridad contra el catálogo de prototipos  
*Describe qué problema o restricción de memoria/sintaxis identificaste al codificar a mano:*
> Esta fue la más corta hasta ahora, pero me hizo pensar en el orden de los `return`. Al principio puse el `return -1` dentro del `for`, en un `else`, y eso estaba mal: si la función que busco no es la primera del catálogo, ya regresaba -1 sin revisar las demás. Lo saqué del `for` para que solo regrese -1 cuando ya revisé todo.
>
> También me confundí un rato entre 0 y -1. El 0 es "sí existe pero le mandaste mal los argumentos" y el -1 es "esa función ni existe". Por eso, en cuanto encuentro el nombre, ya regreso 1 o 0, y el -1 queda hasta el final.
>
> El caso de `main` con 0 argumentos lo tenía medio en duda, pero como comparo `aridad == nargs` y los dos son 0, sale 1 sin hacer nada especial.
>
> Le agregué `catalogo[i].nombre != NULL` antes del `strcmp` por si alguna entrada viene vacía, para que no truene.

---

## 2. Fase 2: El Científico (Ingeniería de Requerimientos C-R-E-O)

### 2.1 Matriz de Especificación Formal
| Elemento C-R-E-O | Definición Técnica |
| :--- | :--- |
| **Contexto (C)** | C99 / GCC o Rust 2021 sin bibliotecas no estándar |
| **Rol (R)** | Ingeniero de software de sistemas y compiladores |
| **Especificación (E)** | Búsqueda exacta del nombre en todo el catálogo y comparación de aridad (1 / 0 / -1) |
| **Output (O)** | Solo código modular y análisis de complejidad |

### 2.2 Prompt Estructurado
```text
CONTEXTO: Laboratorio de compiladores en C (C11, GCC) sobre verificación de
llamadas a funciones. Se compila con gcc -std=c11 -Wall -Wextra -Werror, sin
warnings. Solo librerías estándar y sin main() porque el harness ya tiene uno.

ROL: Actúa como ingeniero de software de sistemas y compiladores.

ESPECIFICACIÓN: Respeta exactamente esto de api_sesion16.h:

typedef struct Funcion {
    const char *nombre;
    int aridad;
} Funcion;

int verificar_llamada(const Funcion *catalogo, int total, const char *nombre,
                      int nargs);

Reglas:
- Busca el nombre en TODO el catálogo (0 a total-1) con strcmp exacto.
- Si lo encuentra y aridad == nargs regresa 1; si la aridad no coincide regresa 0.
- Si no está en ningún lado regresa -1 (solo después de revisar todo).
- Compara nargs contra la aridad de la función, no contra total.
- Aridad 0 solo acepta 0 argumentos.
- catalogo NULL, nombre NULL o nargs negativo regresan -1.

OUTPUT: Solo el código de ia_sesion16_c.c, corto, y una línea con la complejidad.
```

---

## 3. Código Devuelto por el Modelo
```c
#include "api_sesion16.h"
#include <string.h>

int verificar_llamada(const Funcion *catalogo, int total, const char *nombre, int nargs) {
    if (!catalogo || !nombre || nargs < 0 || total < 0) return -1;

    for (int i = 0; i < total; i++) {
        if (catalogo[i].nombre && strcmp(catalogo[i].nombre, nombre) == 0) {
            return catalogo[i].aridad == nargs ? 1 : 0;
        }
    }
    return -1; // no esta registrada
}

// Complejidad: O(n), n = funciones en el catalogo
```

---

## 4. Fase 3: El Auditor (Resultados del Test Harness)
*Pruebas ejecutadas:* Aridad correcta, de menos, de más, función inexistente, aridad 0 y nombre NULL

### Salida del harness
```text
=== AUDITORIA SESION 16 - Registro de prototipos y aridad ===
Caso                               esperado:              obtenido:              resultado
[ 1] suma con 3 argumentos              esperado: 1            obtenido: 1            OK
[ 2] suma con 2 argumentos              esperado: 0            obtenido: 0            OK
[ 3] suma con 4 argumentos              esperado: 0            obtenido: 0            OK
[ 4] funcion no registrada              esperado: -1           obtenido: -1           OK
[ 5] main sin argumentos                esperado: 1            obtenido: 1            OK
[ 6] nombre NULL                        esperado: -1           obtenido: -1           OK

RESULTADO: 6/6 pruebas superadas
TODO EN VERDE. Documenta tu auditoria en auditorias/auditoria_sesion16.md
```

* [x] Caso base y límites superados sin *Crash*.
* [x] No existen accesos fuera de límites ni *Memory Leaks*.
* [x] La complejidad teórica coincide con la implementada.

### Pruebas extra que hice yo
Catálogo de 4 funciones: `suma(3)`, `resta(2)`, `imprimir(1)`, `main(0)`

| Prueba | Esperado | Obtenido |
| :--- | :--- | :--- |
| `resta` con 2 (está en la posición 2) | 1 | 1 |
| `main` con 0 (está en la última posición) | 1 | 1 |
| `main` con 1 | 0 | 0 |
| `sum` con 3 (parecido a "suma") | -1 | -1 |
| `suma` con -1 argumentos | -1 | -1 |
| catálogo NULL | -1 | -1 |

### Reporte Crítico
*¿Qué caso borde o problema omitió la IA y cómo lo solucionaste?*
> En la primera versión de la IA, el `for` tenía un `if` con `else return -1;`. O sea, si la primera función del catálogo no era la que buscaba, regresaba -1 de una vez y no revisaba las demás. En el harness eso no se nota, porque todos los casos usan un catálogo de **una sola función**, y salía 6/6.
>
> Lo detecté porque a mí me pasó lo mismo en la Fase 1. Hice una prueba con un catálogo de 4 funciones y busqué `resta`, que está en la segunda posición: la versión de la IA me regresaba -1 cuando debía ser 1.
>
> Lo arreglé poniendo en el contrato que tiene que buscar en **todo** el catálogo y que el -1 va solo después de revisar todo. La segunda versión ya tiene el `return -1` afuera del `for`.
>
> Conclusión: si el harness solo prueba con un elemento no te das cuenta de que el ciclo está mal. Hay que probar con varios.

---
**Firma digital:** ReyesDiego002
