# Bitácora de Requerimientos y Auditoría IA — Sesión 09
**Materia:** Lenguajes y Autómatas II (SCD-1016)  
**Tema:** Reporte Formal de Errores Semánticos y Diagnóstico  
**Docente Titular:** MCC. Iván Márquez Larios (`ijmarquezl`)  
**Estudiante:** Diego Reyes Martínez  
**Matrícula:** 24531376  
**Track:** C (el harness evalúa C; la versión en Rust es opcional y no se califica con este harness)  

---

## 1. Fase 1: El Artesano (Dificultad Manual)
*Reto:* Formato de diagnóstico estilo GCC sin desbordar el buffer  
*Describe qué problema o restricción de memoria/sintaxis identificaste al codificar a mano:*
> Lo primero que pensé fue usar `sprintf`, pero `sprintf` no sabe el tamaño del buffer. En el caso 4 el buffer es de solo 10 bytes y el mensaje completo mide 37 caracteres, así que se escribiría fuera del arreglo (desbordamiento). Por eso usé `snprintf`, que recibe el tamaño, escribe como máximo `cap-1` caracteres y siempre pone el `'\0'`.
>
> También me atoré con el tipo de `cap`: es `int`, pero `snprintf` pide `size_t`. Si `cap` fuera negativo, al convertirlo daría un número enorme. Por eso primero reviso `cap <= 0` y luego hago el cast `(size_t)cap`.
>
> Otro detalle fue el formato exacto: el harness compara carácter por carácter, así que no puede llevar espacios de más, punto final ni `\n`.

---

## 2. Fase 2: El Científico (Ingeniería de Requerimientos C-R-E-O)

### 2.1 Matriz de Especificación Formal
| Elemento C-R-E-O | Definición Técnica |
| :--- | :--- |
| **Contexto (C)** | C99 / GCC o Rust 2021 sin bibliotecas no estándar |
| **Rol (R)** | Ingeniero de software de sistemas y compiladores |
| **Especificación (E)** | Diagnóstico `archivo:línea:columna: error: mensaje` con truncamiento seguro |
| **Output (O)** | Solo código modular y análisis de complejidad |

### 2.2 Prompt Estructurado
```text
CONTEXTO: Estoy en un laboratorio de compiladores en C (C11, GCC) sin bibliotecas
no estándar. Se compila con: gcc -std=c11 -Wall -Wextra -Werror, así que no puede
tener advertencias. Mi archivo se compila junto con un harness que ya trae main(),
por eso NO debe tener main(). El buffer de salida lo reserva quien llama.

ROL: Actúa como ingeniero de software de sistemas y compiladores que conoce el
formato de diagnósticos de GCC.

ESPECIFICACIÓN: Implementa esta función respetando EXACTAMENTE la firma de
api_sesion09.h:

void reportar(const char *archivo, int linea, int columna, const char *mensaje,
              char *salida, int cap);

Reglas:
- Escribe en salida: archivo:linea:columna: error: mensaje
- Formato exacto: sin espacios extra, sin punto final, sin salto de línea.
- Nunca escribas más de cap-1 caracteres; el resultado siempre termina en '\0'.
- Si salida es NULL o cap <= 0, no escribas nada.
- Línea y columna pueden ser 0.
- Ejemplo: reportar("main.c",14,5,"tipo incompatible",b,10) deja "main.c:14".

OUTPUT: Solo el código del archivo ia_sesion09_c.c y al final una línea con la
complejidad.
```

---

## 3. Código Devuelto por el Modelo
```c
#include "api_sesion09.h"
#include <stdio.h>

void reportar(const char *archivo, int linea, int columna, const char *mensaje, char *salida, int cap) {
    int n;

    if (salida == NULL || cap <= 0) {
        return; /* no hay donde escribir */
    }
    if (archivo == NULL) archivo = "(desconocido)";
    if (mensaje == NULL) mensaje = "";

    /* formato estilo GCC: archivo:linea:columna: error: mensaje */
    n = snprintf(salida, (size_t)cap, "%s:%d:%d: error: %s",
                 archivo, linea, columna, mensaje);

    if (n < 0) {
        salida[0] = '\0'; /* error de formato: dejamos cadena vacia */
    }
    /* si n >= cap el mensaje se trunco, pero snprintf ya puso el '\0' */
}

/* Complejidad: O(n), n = largo del texto generado (limitado por cap). Memoria O(1). */
```

---

## 4. Fase 3: El Auditor (Resultados del Test Harness)
*Pruebas ejecutadas:* Formato estilo GCC, línea/columna en 0 y truncamiento con buffer chico

### Salida del harness
```text
=== AUDITORIA SESION 09 - Diagnostico con linea y columna ===
Caso                               esperado:              obtenido:              resultado
[ 1] main.c:14:5
     esperado: main.c:14:5: error: tipo incompatible
     obtenido: main.c:14:5: error: tipo incompatible
     -> OK
[ 2] a.c:1:1
     esperado: a.c:1:1: error: x
     obtenido: a.c:1:1: error: x
     -> OK
[ 3] linea 0
     esperado: x.c:0:0: error: y
     obtenido: x.c:0:0: error: y
     -> OK
[ 4] cap=10 trunca                      esperado: main.c:14    obtenido: main.c:14    OK

RESULTADO: 4/4 pruebas superadas
TODO EN VERDE. Documenta tu auditoria en auditorias/auditoria_sesion09.md
```

* [x] Caso base y límites superados sin *Crash*.
* [x] No existen accesos fuera de límites ni *Memory Leaks*.
* [x] La complejidad teórica coincide con la implementada.

### Reporte Crítico
*¿Qué caso borde o problema omitió la IA y cómo lo solucionaste?*
> La primera propuesta de la IA usaba `sprintf(salida, "%s:%d:%d: error: %s", ...)` sin tomar en cuenta `cap`. Los casos 1 a 3 salían OK porque esos buffers son grandes (128 y 64 bytes). En el caso 4, con un buffer de 10 bytes, `sprintf` escribía los 37 caracteres completos fuera del arreglo y el programa terminaba con `*** stack smashing detected ***` o con basura en la salida.
>
> Lo detecté porque el caso 4 del harness falló, y al leer el código vi que `cap` nunca se usaba. De hecho, con `-Wextra` eso debía salir como advertencia de "parámetro sin usar", y la IA lo había tapado con `(void)cap;`.
>
> Lo solucioné pidiéndole que usara `snprintf` con `(size_t)cap`, que revisara `salida == NULL` y `cap <= 0` antes de escribir, y que no dejara el `(void)cap`. También le pedí que aceptara `archivo` o `mensaje` en NULL, un caso que el harness no prueba. Lo comprobé yo con una prueba extra: `reportar(NULL, 1, 1, NULL, b, 64)`.
>
> Conclusión: que pasen los casos con buffers grandes no garantiza que la función sea segura. El caso que de verdad mide la seguridad es el de truncamiento.

---
**Firma digital:** ReyesDiego002
