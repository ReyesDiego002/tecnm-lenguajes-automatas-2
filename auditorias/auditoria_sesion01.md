# Bitácora de Requerimientos y Auditoría IA — Sesión 01
**Materia:** Lenguajes y Autómatas II (SCD-1016)  
**Tema:** Encuadre, Diagnóstico de Punteros y Metodología IA  
**Docente Titular:** MCC. Iván Márquez Larios (`ijmarquezl`)  
**Estudiante:** Diego Reyes Martínez
**Matrícula:** 24531376  
**Track:** [ C ]  

---

## 1. Fase 1: El Artesano (Dificultad Manual)
*Reto:* Inversión de buffer in-place con punteros  
*Describe qué problema o restricción de memoria/sintaxis identificaste al codificar a mano:*
> Lo que más cuidado me pidió fue no salirme del buffer. Para el índice final usé `strlen(s) - 1`, pero me di cuenta de que si uso `size_t` (que no tiene negativos), en una cadena vacía ese cálculo da un número gigante y el programa leería memoria que no es suya. Lo resolví usando `int`, así `fin` queda en -1 y el `while (inicio < fin)` no se ejecuta. También tuve que revisar `NULL` al principio con un `return`, porque llamar a `strlen(NULL)` provoca un fallo de segmentación. Otro detalle fue el intercambio: necesitaba el `char temporal`, porque si asigno directo `s[inicio] = s[fin]` pierdo el carácter original y la cadena queda con letras repetidas. Mi primera versión (el `TODO` vacío) pasaba solo 3 de 6 casos, y fueron los que no cambian la cadena (vacía, un carácter y NULL).

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
Contexto: Estoy en la materia Lenguajes y Autómatas II. Trabajo en C (estándar C11) y compilo con gcc -std=c11 -Wall -Wextra -Werror, así que cualquier advertencia cuenta como error. Ya existe un archivo api_sesion01.h con esta firma:
void invertir(char *s);

Rol: Actúa como ingeniero de software de sistemas y compiladores, con experiencia en manejo de memoria y punteros en C.

Especificación:
- invertir modifica la cadena en su mismo lugar, sin devolver una copia.
- No uses malloc ni arreglos auxiliares: solo punteros o índices y un char temporal.
- invertir(NULL) debe terminar sin caerse.
- La cadena vacía y la cadena de un solo carácter deben quedar igual.
- No uses main() ni cambies la firma.

Output: Dame solo el código del archivo ia_sesion01_c.c con comentarios breves, y al final un análisis corto de la complejidad en tiempo y en espacio.
```

---

## 3. Código Devuelto por el Modelo
```c
// Código devuelto por el LLM
#include "api_sesion01.h"
#include <string.h>

void invertir(char *s) {
    if (s == NULL) {
        return;
    }

    /* si la cadena esta vacia no hay nada que invertir */
    if (*s == '\0') {
        return;
    }

    char *inicio = s;                /* apunta al primer caracter */
    char *fin = s + strlen(s) - 1;   /* apunta al ultimo caracter */
    char temporal;

    while (inicio < fin) {
        temporal = *inicio;
        *inicio = *fin;
        *fin = temporal;
        inicio++;
        fin--;
    }
}

/* Complejidad: tiempo O(n), porque se recorre la mitad de la cadena
   (n/2 intercambios). Espacio O(1), solo se usan dos punteros y un char. */
```

---

## 4. Fase 3: El Auditor (Resultados del Test Harness)
*Pruebas ejecutadas:* Detección de errores de compilación GCC

* [x] Caso base y límites superados sin *Crash*.
* [x] No existen accesos fuera de límites ni *Memory Leaks*.
* [x] La complejidad teórica coincide con la implementada.

Salida del harness:
```text
[ 1] invertir("HOLA")                   esperado: ALOH         obtenido: ALOH         OK
[ 2] invertir("abc")                    esperado: cba          obtenido: cba          OK
[ 3] invertir("god")                    esperado: dog          obtenido: dog          OK
[ 4] invertir("AB")                     esperado: BA           obtenido: BA           OK
[ 5] invertir("a")                      esperado: a            obtenido: a            OK
[ 6] invertir(NULL)                     esperado: sin crash    obtenido: sin crash    OK

RESULTADO: 6/6 pruebas superadas
TODO EN VERDE. Documenta tu auditoria en auditorias/auditoria_sesion01.md
```

### Reporte Crítico
*¿Qué caso borde o problema omitió la IA y cómo lo solucionaste?*
> En su primera respuesta la IA calculaba `s + strlen(s) - 1` directo, sin revisar antes si la cadena estaba vacía. Con una cadena vacía, `fin` quedaba apuntando un byte antes del inicio del buffer, o sea fuera de los límites, y aunque el `while` no entraba, ese puntero ya era inválido. Lo detecté al revisar el código con el caso `invertir("")` en mente y pensando qué valor tomaba `fin`. Lo corregí agregando una condición `if (*s == '\0') return;` antes de calcular `fin`. También verifiqué que no usara `malloc` ni arreglos auxiliares, y que la firma fuera idéntica a la de `api_sesion01.h`. Después compilé con `-Wall -Wextra -Werror` y el harness salió 6/6 en verde.

---
**Firma digital:** [ReyesDiego002]
