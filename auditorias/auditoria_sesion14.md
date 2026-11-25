# Bitácora de Requerimientos y Auditoría IA — Sesión 14
**Materia:** Lenguajes y Autómatas II (SCD-1016)  
**Tema:** Declaración vs Uso de Variables  
**Docente Titular:** MCC. Iván Márquez Larios (`ijmarquezl`)  
**Estudiante:** Diego Reyes Martínez  
**Matrícula:** 24531376  
**Track:** C (el harness evalúa C; la versión en Rust es opcional y no se califica con este harness)  

---

## 1. Fase 1: El Artesano (Dificultad Manual)
*Reto:* Regla de declaración previa y detección de redeclaración  
*Describe qué problema o restricción de memoria/sintaxis identificaste al codificar a mano:*
> Esta estuvo más tranquila que la de ámbitos, porque es una sola tabla. Lo que se me ocurrió fue reusar `usar()` dentro de `declarar()`: si `usar` dice que ya existe, regreso 0 y no lo meto. Así no escribo el mismo `for` dos veces.
>
> El problema fue que en el archivo `usar` está escrita después de `declarar`. Pensé que iba a dar error, pero no, porque `usar` ya viene declarada en el `.h`, que incluyo arriba.
>
> En lo que sí me fijé fue en la comparación. Si usara `strncmp` con el largo del nombre guardado, `"x"` y `"xy"` saldrían iguales porque las dos empiezan con "x". Por eso usé `strcmp`, que compara todo hasta el `'\0'`.
>
> Y como el arreglo es de 16, antes de guardar reviso que no esté lleno. Si no lo hiciera, me saldría del arreglo.

---

## 2. Fase 2: El Científico (Ingeniería de Requerimientos C-R-E-O)

### 2.1 Matriz de Especificación Formal
| Elemento C-R-E-O | Definición Técnica |
| :--- | :--- |
| **Contexto (C)** | C99 / GCC o Rust 2021 sin bibliotecas no estándar |
| **Rol (R)** | Ingeniero de software de sistemas y compiladores |
| **Especificación (E)** | Declarar sin duplicados y usar con comparación exacta de nombres |
| **Output (O)** | Solo código modular y análisis de complejidad |

### 2.2 Prompt Estructurado
```text
CONTEXTO: Laboratorio de compiladores en C (C11, GCC) sobre la regla de declarar
antes de usar. Se compila con gcc -std=c11 -Wall -Wextra -Werror, sin warnings.
Solo librerías estándar y sin main() porque el harness ya tiene uno.

ROL: Actúa como ingeniero de software de sistemas y compiladores.

ESPECIFICACIÓN: Respeta exactamente esto de api_sesion14.h:

#define TABLA_MAX 16
typedef struct Tabla {
    const char *nombres[TABLA_MAX];
    int total;
} Tabla;

int declarar(Tabla *tabla, const char *nombre);
int usar(const Tabla *tabla, const char *nombre);

Reglas:
- declarar: 1 si insertó, 0 si ya estaba (sin cambiar total), -1 si tabla o
  nombre son NULL, si el nombre es "" o si la tabla está llena.
- usar: 1 solo si el nombre ya fue declarado, 0 en cualquier otro caso.
- Comparación exacta con strcmp: "x" y "xy" son distintos. No uses strncmp.

OUTPUT: Solo el código de ia_sesion14_c.c, corto, y una línea con la complejidad.
```

---

## 3. Código Devuelto por el Modelo
```c
#include "api_sesion14.h"
#include <string.h>

int usar(const Tabla *tabla, const char *nombre) {
    if (!tabla || !nombre) return 0;
    for (int i = 0; i < tabla->total; i++) {
        if (strcmp(tabla->nombres[i], nombre) == 0) return 1; // igual exacto
    }
    return 0;
}

int declarar(Tabla *tabla, const char *nombre) {
    if (!tabla || !nombre || !*nombre) return -1;
    if (usar(tabla, nombre)) return 0;           // redeclaracion
    if (tabla->total >= TABLA_MAX) return -1;    // ya no cabe
    tabla->nombres[tabla->total++] = nombre;
    return 1;
}

// Complejidad: usar O(n), declarar O(n), n = nombres en la tabla
```

---

## 4. Fase 3: El Auditor (Resultados del Test Harness)
*Pruebas ejecutadas:* Uso sin declarar, uso válido, redeclaración, total, prefijo y nombre vacío

### Salida del harness
```text
=== AUDITORIA SESION 14 - Declaracion previa y redeclaracion ===
Caso                               esperado:              obtenido:              resultado
[ 1] usar sin declarar                  esperado: 0            obtenido: 0            OK
[ 2] usar despues de declarar           esperado: 1            obtenido: 1            OK
[ 3] redeclarar el mismo nombre         esperado: 0            obtenido: 0            OK
[ 4] total tras redeclarar              esperado: 1            obtenido: 1            OK
[ 5] prefijo no es el mismo nombre      esperado: 0            obtenido: 0            OK
[ 6] declarar nombre vacio              esperado: -1           obtenido: -1           OK

RESULTADO: 6/6 pruebas superadas
TODO EN VERDE. Documenta tu auditoria en auditorias/auditoria_sesion14.md
```

* [x] Caso base y límites superados sin *Crash*.
* [x] No existen accesos fuera de límites ni *Memory Leaks*.
* [x] La complejidad teórica coincide con la implementada.

### Pruebas extra que hice yo
| Prueba | Esperado | Obtenido |
| :--- | :--- | :--- |
| Declarar `"x"` y `"xy"`, luego usar las dos | 1 y 1, total 2 | 1 y 1, total 2 |
| `declarar(NULL, "a")` | -1 | -1 |
| `usar(NULL, "x")` | 0 | 0 |
| Declarar 20 nombres distintos | se para en 16, el resto -1 | total 16, último -1 |

### Reporte Crítico
*¿Qué caso borde o problema omitió la IA y cómo lo solucionaste?*
> En la primera versión que me dio la IA (cuando no le dije nada de cómo comparar), `usar` hacía `strncmp(tabla->nombres[i], nombre, strlen(tabla->nombres[i]))`. O sea, comparaba solo el largo del nombre guardado. Con eso, si declarabas `"x"` y luego usabas `"xy"`, decía que sí existía, porque compara solo la primera letra.
>
> Esta vez sí lo detectó el harness: el caso 5 (`prefijo no es el mismo nombre`) salió FALLA y me dio 5/6. Al revisar el código vi el `strncmp`.
>
> Lo arreglé poniendo en el contrato que la comparación tiene que ser exacta con `strcmp`, con el ejemplo de `"x"` y `"xy"`. También le pedí que revisara si la tabla está llena, porque la primera versión tampoco lo hacía y el harness eso no lo prueba. Lo probé yo declarando 20 nombres.
>
> Conclusión: comparar cadenas parece fácil, pero un `strncmp` mal usado hace que el compilador acepte variables que nunca se declararon.

---
**Firma digital:** ReyesDiego002463
