# Bitácora de Requerimientos y Auditoría IA — Sesión 18
**Materia:** Lenguajes y Autómatas II (SCD-1016)  
**Tema:** Arreglos y Verificación de Límites Estáticos  
**Docente Titular:** MCC. Iván Márquez Larios (`ijmarquezl`)  
**Estudiante:** Diego Reyes Martínez  
**Matrícula:** 24531376  
**Track:** C (el harness evalúa C; la versión en Rust es opcional y no se califica con este harness)  

---

## 1. Fase 1: El Artesano (Dificultad Manual)
*Reto:* Tamaño de un arreglo en bytes y validación de índices  
*Describe qué problema o restricción de memoria/sintaxis identificaste al codificar a mano:*
> La tabla de anchos ya la tenía de la sesión 12, así que la reusé casi igual.
>
> En lo que más me fijé fue en el índice. Mi primera idea fue `indice <= elementos`, pero eso está mal: un arreglo de 5 va de 0 a 4, y el 5 ya está afuera. Lo cambié a `<`. Justo el caso 5 del harness (`indice 5 de 5`) prueba eso.
>
> Lo otro fue la multiplicación. La función regresa `long`, pero si hago `elementos * a` con los dos `int`, primero multiplica en `int` y luego lo convierte, y si el número es muy grande ya se desbordó. Por eso puse `(long)elementos * a`, para que multiplique directo en `long`.
>
> Con los negativos no tuve problema: `indice >= 0` va primero en el `&&`, así que si es negativo ni siquiera compara con `elementos`.

---

## 2. Fase 2: El Científico (Ingeniería de Requerimientos C-R-E-O)

### 2.1 Matriz de Especificación Formal
| Elemento C-R-E-O | Definición Técnica |
| :--- | :--- |
| **Contexto (C)** | C99 / GCC o Rust 2021 sin bibliotecas no estándar |
| **Rol (R)** | Ingeniero de software de sistemas y compiladores |
| **Especificación (E)** | Tamaño = elementos × ancho (en long) e índice válido en 0 ≤ i < elementos |
| **Output (O)** | Solo código modular y análisis de complejidad |

### 2.2 Prompt Estructurado
```text
CONTEXTO: Laboratorio de compiladores en C (C11, GCC) sobre arreglos y límites.
Se compila con gcc -std=c11 -Wall -Wextra -Werror, sin warnings. Solo librerías
estándar y sin main() porque el harness ya tiene uno.

ROL: Actúa como ingeniero de software de sistemas y compiladores.

ESPECIFICACIÓN: Respeta exactamente estas firmas de api_sesion18.h:

long bytes_arreglo(const char *tipo, int elementos);
int indice_en_rango(int indice, int elementos);

Reglas:
- Anchos: char=1, short=2, int=4, float=4, double=8, puntero=8.
- bytes_arreglo regresa elementos * ancho, multiplicando en long (castea antes)
  para que no se desborde con arreglos grandes. 0 elementos = 0 bytes.
  Elementos negativos, tipo NULL o tipo desconocido regresan -1.
- indice_en_rango regresa 1 si 0 <= indice < elementos, si no 0.
  El índice máximo es elementos - 1.

Ejemplos: int[10]=40, double[3]=24, int[-4]=-1, indice 0 de 5 = 1,
indice 5 de 5 = 0, indice -1 de 5 = 0.

OUTPUT: Solo el código de ia_sesion18_c.c, corto, y una línea con la complejidad.
```

---

## 3. Código Devuelto por el Modelo
```c
#include "api_sesion18.h"
#include <string.h>
#include <stddef.h>

long bytes_arreglo(const char *tipo, int elementos) {
    const char *nombres[] = {"char", "short", "int", "float", "double", "puntero"};
    const int anchos[]    = { 1,      2,       4,     4,       8,        8 };

    if (!tipo || elementos < 0) return -1;

    for (int i = 0; i < 6; i++) {
        if (strcmp(tipo, nombres[i]) == 0) {
            return (long)elementos * anchos[i]; // en long para no desbordar
        }
    }
    return -1; // tipo desconocido
}

int indice_en_rango(int indice, int elementos) {
    return (indice >= 0 && indice < elementos) ? 1 : 0;
}

// Complejidad: bytes_arreglo O(1) (6 tipos fijos), indice_en_rango O(1)
```

---

## 4. Fase 3: El Auditor (Resultados del Test Harness)
*Pruebas ejecutadas:* Tamaño de int y double, dimensión negativa, primer índice, índice igual al tamaño e índice negativo

### Salida del harness
```text
=== AUDITORIA SESION 18 - Tamano de arreglo y limites ===
Caso                               esperado:              obtenido:              resultado
[ 1] int[10]                            esperado: 40           obtenido: 40           OK
[ 2] double[3]                          esperado: 24           obtenido: 24           OK
[ 3] int[-4]                            esperado: -1           obtenido: -1           OK
[ 4] indice 0 de 5                      esperado: 1            obtenido: 1            OK
[ 5] indice 5 de 5                      esperado: 0            obtenido: 0            OK
[ 6] indice negativo                    esperado: 0            obtenido: 0            OK

RESULTADO: 6/6 pruebas superadas
TODO EN VERDE. Documenta tu auditoria en auditorias/auditoria_sesion18.md
```

* [x] Caso base y límites superados sin *Crash*.
* [x] No existen accesos fuera de límites ni *Memory Leaks*.
* [x] La complejidad teórica coincide con la implementada.

### Pruebas extra que hice yo
| Prueba | Esperado | Obtenido |
| :--- | :--- | :--- |
| `int[0]` | 0 | 0 |
| `puntero[2]` | 16 | 16 |
| `long[3]` (tipo que no está) | -1 | -1 |
| tipo NULL | -1 | -1 |
| `double[300000000]` | 2400000000 | 2400000000 |
| índice 4 de 5 (el último) | 1 | 1 |
| índice 0 de 0 | 0 | 0 |
| índice -2147483648 (el int más chico) de 5 | 0 | 0 |

### Reporte Crítico
*¿Qué caso borde o problema omitió la IA y cómo lo solucionaste?*
> En la primera versión de la IA, `bytes_arreglo` hacía `return elementos * anchos[i];` sin el cast a `long`. Con los números del harness (10 y 3) da bien y salía 6/6. Pero `elementos` y `anchos[i]` son `int`, así que la multiplicación se hace en `int` y luego se pasa a `long`. Con un arreglo grande, por ejemplo `double[300000000]`, el resultado real es 2,400,000,000, que no cabe en un `int` (el máximo es como 2,147 millones), y daba un número **negativo**.
>
> Lo detecté haciendo una prueba con un arreglo grande y compilando con `-fsanitize=undefined`, que marcó `signed integer overflow`.
>
> Lo arreglé poniendo en el contrato que la multiplicación se haga en `long`, casteando antes de multiplicar. La segunda versión ya trae `(long)elementos * anchos[i]`.
>
> Conclusión: que una función regrese `long` no quiere decir que la cuenta se haga en `long`. Si no casteas antes, el desbordamiento ya pasó.

---
**Firma digital:** ReyesDiego002
