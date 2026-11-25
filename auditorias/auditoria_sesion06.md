# Bitácora de Requerimientos y Auditoría IA — Sesión 06
**Materia:** Lenguajes y Autómatas II (SCD-1016)  
**Tema:** Sistema de Tipos Primitivos y Verificación Semántica  
**Docente Titular:** MCC. Iván Márquez Larios (`ijmarquezl`)  
**Estudiante:** Diego Reyes Martínez
**Matrícula:** 24531376  
**Track:** [ C ]  

---

## 1. Fase 1: El Artesano (Dificultad Manual)
*Reto:* Tabla de compatibilidad de tipos primitivos  
*Describe qué problema o restricción de memoria/sintaxis identificaste al codificar a mano:*
> Lo primero que entendí fue por qué asignar un tipo más ancho a uno más angosto necesita un cast explícito: por ejemplo, si guardo un `float` como 3.7 en un `int`, se pierde la parte decimal y queda en 3, y el compilador quiere que yo lo haga a propósito. En cambio, de `char` a `int` o de `int` a `float` no se pierde nada. Por eso la regla es que el destino sea igual o más ancho que el origen. Lo que más cuidado me pidió fue `TIPO_ERROR`: como vale 0 y es el más bajo de la jerarquía, si solo comparaba con `>=` el caso `compatible(ERROR, ERROR)` daba 1, porque 0 >= 0. Lo arreglé revisando primero si alguno de los dos es `TIPO_ERROR` y regresando 0 antes de comparar. También agregué un `default` en el `switch` de `nombre_tipo`, para que un valor fuera del enum regrese "desconocido" y el compilador no marque advertencia al compilar con `-Werror`.

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
Contexto: Estoy en la materia Lenguajes y Autómatas II. Trabajo en C (estándar C11) y compilo con gcc -std=c11 -Wall -Wextra -Werror, así que cualquier advertencia cuenta como error. Ya existe un archivo api_sesion06.h con este enum y estas firmas:
typedef enum { TIPO_ERROR = 0, TIPO_CHAR = 1, TIPO_INT = 2, TIPO_FLOAT = 3 } Tipo;
int compatible(Tipo destino, Tipo origen);
const char *nombre_tipo(Tipo t);

Rol: Actúa como ingeniero de software de sistemas y compiladores, con experiencia en verificación semántica y sistemas de tipos.

Especificación:
- compatible devuelve 1 si se puede asignar un valor de tipo origen a una variable de tipo destino, y 0 si no. La jerarquía es CHAR < INT < FLOAT: se acepta si el destino es igual o más ancho que el origen. Asignar a un tipo más angosto se rechaza porque requiere cast explícito.
- TIPO_ERROR nunca es compatible con nada, ni como destino ni como origen (incluso compatible(ERROR, ERROR) devuelve 0).
- Valores fuera del enum también se rechazan.
- nombre_tipo devuelve "char", "int", "float" o "error", y "desconocido" para un valor fuera del enum.
- No uses main() ni cambies las firmas ni el enum.

Output: Dame solo el código del archivo ia_sesion06_c.c con comentarios breves, y al final un análisis corto de la complejidad en tiempo y en espacio.
```

---

## 3. Código Devuelto por el Modelo
```c
// Código devuelto por el LLM
#include "api_sesion06.h"
#include <stdio.h>

/* tabla[destino][origen]: 1 si la asignacion es valida, 0 si no
 * columnas y filas en orden: ERROR, CHAR, INT, FLOAT */
static const int tabla[4][4] = {
    /* destino ERROR */ {0, 0, 0, 0},
    /* destino CHAR  */ {0, 1, 0, 0},
    /* destino INT   */ {0, 1, 1, 0},
    /* destino FLOAT */ {0, 1, 1, 1}
};

static const char *nombres[4] = {"error", "char", "int", "float"};

int compatible(Tipo destino, Tipo origen) {
    int d = (int)destino;
    int o = (int)origen;

    /* si algun valor esta fuera de la tabla, lo rechazo */
    if (d < 0 || d > 3 || o < 0 || o > 3) {
        return 0;
    }
    return tabla[d][o];
}

const char *nombre_tipo(Tipo t) {
    int i = (int)t;

    if (i < 0 || i > 3) {
        return "desconocido";
    }
    return nombres[i];
}

/* Complejidad: ambas funciones son O(1) en tiempo, porque solo hacen
   comparaciones y un acceso directo a un arreglo. Espacio O(1): la tabla
   y los nombres son de tamano fijo. */
```

---

## 4. Fase 3: El Auditor (Resultados del Test Harness)
*Pruebas ejecutadas:* Detección de errores de compilación GCC

* [x] Caso base y límites superados sin *Crash*.
* [x] No existen accesos fuera de límites ni *Memory Leaks*.
* [x] La complejidad teórica coincide con la implementada.

Salida del harness:
```text
[ 1] compatible(INT,INT)                esperado: 1            obtenido: 1            OK
[ 2] compatible(FLOAT,INT)              esperado: 1            obtenido: 1            OK
[ 3] compatible(INT,FLOAT)              esperado: 0            obtenido: 0            OK
[ 4] compatible(INT,ERROR)              esperado: 0            obtenido: 0            OK
[ 5] nombre_tipo(INT)                   esperado: int          obtenido: int          OK
[ 6] nombre_tipo(ERROR)                 esperado: error        obtenido: error        OK

RESULTADO: 6/6 pruebas superadas
```

### Reporte Crítico
*¿Qué caso borde o problema omitió la IA y cómo lo solucionaste?*
> En su primera respuesta la IA resolvía `compatible` con una sola comparación (`destino >= origen`). Eso pasaba los 6 casos del harness, pero fallaba con `compatible(TIPO_ERROR, TIPO_ERROR)`, que devolvía 1 porque 0 >= 0, y TIPO_ERROR nunca debe ser compatible con nada. El harness solo prueba `ERROR` como origen, así que ese error no se veía. Lo detecté probando a mano los casos donde destino y origen son iguales. Otro problema era que un valor fuera del enum (por ejemplo, un 7 convertido a `Tipo`) hacía que se leyera fuera de la tabla. Lo corregí cambiando a una tabla de 4x4 donde la fila y la columna de `ERROR` son todas 0, y agregando una revisión del rango (0 a 3) antes de acceder a la tabla. También verifiqué que `nombre_tipo` regresara "desconocido" en vez de salirse del arreglo. Después compilé con `-Wall -Wextra -Werror` y el harness salió 6/6 en verde.

---
**Firma digital:** [ReyesDiego002]
