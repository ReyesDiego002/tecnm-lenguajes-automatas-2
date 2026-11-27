# Bitácora de Requerimientos y Auditoría IA — Sesión 27
**Materia:** Lenguajes y Autómatas II (SCD-1016)  
**Tema:** Indexación de Arreglos en Código Intermedio  
**Docente Titular:** MCC. Iván Márquez Larios (`ijmarquezl`)  
**Estudiante:** Diego Reyes Martínez
**Matrícula:** 24531376  
**Track:** [ C ]  

---

## 1. Fase 1: El Artesano (Dificultad Manual)
*Reto:* Cálculo de offsets e indexación de arreglos 1D y 2D en TAC  
*Describe qué problema o restricción de memoria/sintaxis identificaste al codificar a mano:*
> Lo más difícil fue llevar bien la cuenta de los temporales. Cada línea usa el temporal anterior, así que guardé el número inicial en una variable `t` y armé los nombres con `t`, `t + 1`, `t + 2` y `t + 3`, y solo al final actualicé `*temporales`. En 2D el orden importa: primero `i * filas`, luego se suma `j`, y hasta después se multiplica por el ancho del elemento; si cambio el orden o las dimensiones, se lee otra celda. Otro problema fue el tamaño de las líneas, porque cada una mide 64 caracteres y si el nombre de una variable es muy largo, `sprintf` se pasaría de memoria. Por eso usé `snprintf` con el límite de 64. También validé que `base`, `indice` y los demás punteros no fueran NULL, y que `max` alcanzara para las 2 o las 4 líneas, para no escribir fuera del arreglo `lineas`. Si algo falla regreso 0 sin cambiar `*temporales`.

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
Contexto: Estoy en la materia Lenguajes y Autómatas II. Trabajo en C (estándar C11) y compilo con gcc -std=c11 -Wall -Wextra -Werror, así que cualquier advertencia cuenta como error. Ya existe un archivo api_sesion27.h con estas firmas:
int emitir_acceso(char lineas[][64], int max, const char *base, const char *indice, int ancho, int *temporales);
int emitir_acceso_2d(char lineas[][64], int max, const char *base, const char *i, const char *j, int filas, int ancho, int *temporales);

Rol: Actúa como ingeniero de software de sistemas y compiladores, con experiencia en generación de código intermedio (TAC) y acceso a arreglos.

Especificación:
- emitir_acceso escribe 2 líneas TAC: "_tN = indice * ancho" y "_tN+1 = base[_tN]". Devuelve cuántas líneas escribió.
- emitir_acceso_2d escribe 4 líneas TAC: "_tN = i * filas", "_tN+1 = _tN + j", "_tN+2 = _tN+1 * ancho" y "_tN+3 = base[_tN+2]". Respeta el orden i * filas + j. Devuelve cuántas líneas escribió.
- Los temporales se numeran desde *temporales y cada uno nuevo lo incrementa, sin repetir nombres.
- Si base, indice, i, j, lineas o temporales son NULL, o no caben las líneas en max, o ancho/filas no son positivos, devuelve 0 sin escribir nada ni cambiar *temporales.
- Cada línea mide 64 caracteres: no escribas más de eso (usa snprintf).
- No uses main() ni cambies las firmas.

Output: Dame solo el código del archivo ia_sesion27_c.c con comentarios breves, y al final un análisis corto de la complejidad en tiempo y en espacio.
```

---

## 3. Código Devuelto por el Modelo
```c
// Código devuelto por el LLM
#include "api_sesion27.h"
#include <stdio.h>

/* regresa 1 si snprintf escribio bien, 0 si se corto o fallo */
static int cabio(int n) {
    return n >= 0 && n < 64;
}

int emitir_acceso(char lineas[][64], int max, const char *base, const char *indice, int ancho, int *temporales) {
    if (lineas == NULL || base == NULL || indice == NULL || temporales == NULL) {
        return 0;
    }
    if (max < 2 || ancho <= 0) {
        return 0;
    }

    int t = *temporales;
    int a = snprintf(lineas[0], 64, "_t%d = %s * %d", t, indice, ancho);
    int b = snprintf(lineas[1], 64, "_t%d = %s[_t%d]", t + 1, base, t);

    /* si algun nombre era muy largo y se corto la linea, no cuento nada */
    if (!cabio(a) || !cabio(b)) {
        return 0;
    }

    *temporales = t + 2;
    return 2;
}

int emitir_acceso_2d(char lineas[][64], int max, const char *base, const char *i, const char *j, int filas, int ancho, int *temporales) {
    if (lineas == NULL || base == NULL || i == NULL || j == NULL || temporales == NULL) {
        return 0;
    }
    if (max < 4 || filas <= 0 || ancho <= 0) {
        return 0;
    }

    int t = *temporales;
    int a = snprintf(lineas[0], 64, "_t%d = %s * %d", t, i, filas);
    int b = snprintf(lineas[1], 64, "_t%d = _t%d + %s", t + 1, t, j);
    int c = snprintf(lineas[2], 64, "_t%d = _t%d * %d", t + 2, t + 1, ancho);
    int d = snprintf(lineas[3], 64, "_t%d = %s[_t%d]", t + 3, base, t + 2);

    if (!cabio(a) || !cabio(b) || !cabio(c) || !cabio(d)) {
        return 0;
    }

    *temporales = t + 4;
    return 4;
}

/* Complejidad: tiempo O(1) en ambas funciones, porque emiten un numero fijo
   de lineas (2 y 4). Espacio O(1): solo variables enteras locales; las
   lineas se escriben en el arreglo que ya recibe la funcion. */
```

---

## 4. Fase 3: El Auditor (Resultados del Test Harness)
*Pruebas ejecutadas:* Detección de errores de compilación GCC

* [x] Caso base y límites superados sin *Crash*.
* [x] No existen accesos fuera de límites ni *Memory Leaks*.
* [x] La complejidad teórica coincide con la implementada.

Salida del harness:
```text
[ 1] 1D: i * 4                          esperado: _t0 = i * 4  obtenido: _t0 = i * 4  OK
[ 2] 1D: A[_t0]                         esperado: _t1 = A[_t0] obtenido: _t1 = A[_t0] OK
[ 3] 2D: i * 3                          esperado: _t0 = i * 3  obtenido: _t0 = i * 3  OK
[ 4] 2D: offset por ancho
     esperado: _t2 = _t1 * 4
     obtenido: _t2 = _t1 * 4
     -> OK
[ 5] 2D: M[_t2]                         esperado: _t3 = M[_t2] obtenido: _t3 = M[_t2] OK
[ 6] base NULL (sin acceso)             esperado: 0            obtenido: 0            OK

RESULTADO: 6/6 pruebas superadas
TODO EN VERDE. Documenta tu auditoria en auditorias/auditoria_sesion27.md
```

### Reporte Crítico
*¿Qué caso borde o problema omitió la IA y cómo lo solucionaste?*
> En su primera respuesta la IA escribía las líneas directamente y revisaba `max` hasta el final, o ni lo revisaba. Si `max` era menor a 4 en el acceso 2D, escribía fuera del arreglo `lineas`. Además usaba `sprintf` sin límite, así que un nombre de variable muy largo pasaba los 64 caracteres de la línea, y actualizaba `*temporales` aunque no hubiera emitido nada, así que se saltaban números de temporales. El harness no prueba esos casos porque siempre usa arreglos grandes y nombres cortos, así que lo detecté revisando el código y pensando qué pasaba con `max` chico. Lo corregí validando `max`, los punteros NULL y los valores de `ancho` y `filas` antes de escribir, usando `snprintf` con el límite de 64 y revisando que la línea no se cortara, y actualizando `*temporales` solo al final y solo si todo salió bien. También verifiqué el orden `i * filas + j` y que cada acceso multiplicara por el ancho. Después compilé con `-Wall -Wextra -Werror` y el harness salió 6/6 en verde.

---
**Firma digital:** [ReyesDiego002]
