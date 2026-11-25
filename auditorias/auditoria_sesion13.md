# Bitácora de Requerimientos y Auditoría IA — Sesión 13
**Materia:** Lenguajes y Autómatas II (SCD-1016)  
**Tema:** Manejo de Ámbitos (Scope Global vs Local)  
**Docente Titular:** MCC. Iván Márquez Larios (`ijmarquezl`)  
**Estudiante:** Diego Reyes Martínez  
**Matrícula:** 24531376  
**Track:** C (el harness evalúa C; la versión en Rust es opcional y no se califica con este harness)  

---

## 1. Fase 1: El Artesano (Dificultad Manual)
*Reto:* Pila de ámbitos enlazados y búsqueda hacia la raíz  
*Describe qué problema o restricción de memoria/sintaxis identificaste al codificar a mano:*
> Al principio pensé que `declarar` tenía que revisar también en el papá para no repetir nombres, pero eso rompía el shadowing: si `x` ya estaba en global, ya no me dejaba declarar otra `x` en el bloque. Volví a leer el contrato y dice que `declarar` solo revisa el ámbito actual, así que lo dejé así.
>
> En `buscar` me confundí con qué regresar. Primero regresaba el nombre de la variable (`"x"`), pero el harness espera el nombre del ámbito (`"global"`, `"bloque"`). Lo cambié a `return ambito->nombre`.
>
> Para subir de nivel usé un `while (ambito != NULL)` y al final `ambito = ambito->padre`. Como el global tiene `padre = NULL`, ahí se para solo.
>
> También me acordé de que el arreglo solo tiene 8 lugares, así que antes de guardar reviso `total >= AMBITO_MAX`. Si no, me salgo del arreglo.

---

## 2. Fase 2: El Científico (Ingeniería de Requerimientos C-R-E-O)

### 2.1 Matriz de Especificación Formal
| Elemento C-R-E-O | Definición Técnica |
| :--- | :--- |
| **Contexto (C)** | C99 / GCC o Rust 2021 sin bibliotecas no estándar |
| **Rol (R)** | Ingeniero de software de sistemas y compiladores |
| **Especificación (E)** | Declarar solo en el ámbito actual y buscar del más interno hacia la raíz (shadowing) |
| **Output (O)** | Solo código modular y análisis de complejidad |

### 2.2 Prompt Estructurado
```text
CONTEXTO: Laboratorio de compiladores en C (C11, GCC) sobre ámbitos global y local.
Se compila con gcc -std=c11 -Wall -Wextra -Werror, sin warnings. Solo librerías
estándar y sin main() porque el harness ya tiene uno.

ROL: Actúa como ingeniero de software de sistemas y compiladores.

ESPECIFICACIÓN: Respeta exactamente esto de api_sesion13.h:

#define AMBITO_MAX 8
typedef struct Ambito {
    const char *nombre;
    const char *nombres[AMBITO_MAX];
    int total;
    struct Ambito *padre;
} Ambito;

int declarar(Ambito *ambito, const char *nombre);
const char *buscar(const Ambito *ambito, const char *nombre);

Reglas:
- declarar guarda el nombre solo en ese ámbito, NO revisa los padres.
  Regresa 1 si guardó, 0 si ya estaba en ese ámbito, -1 si el ámbito está
  lleno (8 nombres), si ambito o nombre son NULL o si el nombre es "".
  Nunca escribas más allá de nombres[7].
- buscar empieza en el ámbito actual y sube por padre hasta la raíz.
  Regresa el nombre del ÁMBITO donde lo encontró, o NULL si no existe.

OUTPUT: Solo el código de ia_sesion13_c.c, corto, y una línea con la complejidad.
```

---

## 3. Código Devuelto por el Modelo
```c
#include "api_sesion13.h"
#include <string.h>

// busca el nombre solo dentro de un ambito
static int esta_en(const Ambito *a, const char *nombre) {
    for (int i = 0; i < a->total; i++) {
        if (strcmp(a->nombres[i], nombre) == 0) return 1;
    }
    return 0;
}

int declarar(Ambito *ambito, const char *nombre) {
    if (!ambito || !nombre || !*nombre) return -1;
    if (esta_en(ambito, nombre)) return 0;
    if (ambito->total >= AMBITO_MAX) return -1; // lleno
    ambito->nombres[ambito->total++] = nombre;
    return 1;
}

const char *buscar(const Ambito *ambito, const char *nombre) {
    if (!nombre) return NULL;
    for (const Ambito *a = ambito; a != NULL; a = a->padre) {
        if (esta_en(a, nombre)) return a->nombre;
    }
    return NULL;
}

// Complejidad: declarar O(n), buscar O(n * h) (n = nombres por ambito, h = niveles)
```

---

## 4. Fase 3: El Auditor (Resultados del Test Harness)
*Pruebas ejecutadas:* Declarar, búsqueda local, subir al padre, shadowing, tres niveles y nombre inexistente

### Salida del harness
```text
=== AUDITORIA SESION 13 - Pila de ambitos y busqueda jerarquica ===
Caso                               esperado:              obtenido:              resultado
[ 1] declarar(&g, "x")                  esperado: 1            obtenido: 1            OK
[ 2] buscar en el mismo ambito          esperado: global       obtenido: global       OK
[ 3] buscar subiendo al padre           esperado: global       obtenido: global       OK
[ 4] shadowing en el hijo               esperado: bloque       obtenido: bloque       OK
[ 5] tres niveles hasta la raiz         esperado: raiz         obtenido: raiz         OK
[ 6] nombre no declarado                esperado: (nulo)       obtenido: (nulo)       OK

RESULTADO: 6/6 pruebas superadas
TODO EN VERDE. Documenta tu auditoria en auditorias/auditoria_sesion13.md
```

* [x] Caso base y límites superados sin *Crash*.
* [x] No existen accesos fuera de límites ni *Memory Leaks*.
* [x] La complejidad teórica coincide con la implementada.

### Pruebas extra que hice yo
| Prueba | Esperado | Obtenido |
| :--- | :--- | :--- |
| Declarar 9 nombres en el mismo ámbito | 8 veces 1, el noveno -1 | 1 1 1 1 1 1 1 1 -1 |
| Declarar `"a"` otra vez | 0 | 0 |
| Declarar `""` | -1 | -1 |
| `declarar(NULL, "x")` | -1 | -1 |
| `buscar(NULL, "a")` | NULL | NULL |

### Reporte Crítico
*¿Qué caso borde o problema omitió la IA y cómo lo solucionaste?*
> En la primera versión, la IA no revisaba si el ámbito estaba lleno: hacía `ambito->nombres[ambito->total++] = nombre;` directo. Con los casos del harness no pasa nada, porque nunca declara más de un nombre por ámbito, y salía 6/6. Pero si declaras 9 nombres, el noveno se escribe fuera del arreglo y se come el campo `total` o el puntero `padre`.
>
> Lo vi porque en mi versión del Artesano ya tenía el `if (total >= AMBITO_MAX)`. Hice una prueba metiendo 9 nombres y compilé con `-fsanitize=address`, y me marcó el desbordamiento.
>
> Lo arreglé poniendo en el contrato que con 8 nombres regrese -1 y que nunca escriba más allá de `nombres[7]`. Ya con eso la segunda versión trae la validación.
>
> Conclusión: el harness no prueba el límite del arreglo, así que hay que probarlo uno mismo.

---
**Firma digital:** ReyesDiego002
