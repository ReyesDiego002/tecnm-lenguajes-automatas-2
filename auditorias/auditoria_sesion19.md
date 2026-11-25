# Bitácora de Requerimientos y Auditoría IA — Sesión 19
**Materia:** Lenguajes y Autómatas II (SCD-1016)  
**Tema:** Estructuras (struct) y Offsets de Miembros  
**Docente Titular:** MCC. Iván Márquez Larios (`ijmarquezl`)  
**Estudiante:** Diego Reyes Martínez  
**Matrícula:** 24531376  
**Track:** C (el harness evalúa C; la versión en Rust es opcional y no se califica con este harness)  

---

## 1. Fase 1: El Artesano (Dificultad Manual)
*Reto:* Offsets de campos con padding y tamaño total del struct  
*Describe qué problema o restricción de memoria/sintaxis identificaste al codificar a mano:*
> Esta se parece mucho a la sesión 12 (offsets alineados), así que reusé la idea de alinear el cursor al múltiplo del ancho. Hice una función `alinear()` para no repetir la misma cuenta en las dos funciones.
>
> Me atoré con el orden en `offset_campo`: primero tengo que alinear y **después** comparar el nombre, porque el offset que quiero es la posición ya acomodada. Al principio comparaba antes de alinear y en `{char a, int b}` me daba 1 en vez de 4.
>
> Lo que más trabajo me costó entender fue el padding del final. En `{double a, char b}` la suma da 9, pero el harness espera 16. Es porque si haces un arreglo de ese struct, el siguiente `double` tiene que quedar en múltiplo de 8. Por eso guardo el campo más grande (`mayor`) y al final redondeo a ese múltiplo.
>
> El struct vacío lo puse aparte con `if (total == 0) return 0;`. Si no, `mayor` empieza en 1 y igual da 0, pero así se entiende mejor.

---

## 2. Fase 2: El Científico (Ingeniería de Requerimientos C-R-E-O)

### 2.1 Matriz de Especificación Formal
| Elemento C-R-E-O | Definición Técnica |
| :--- | :--- |
| **Contexto (C)** | C99 / GCC o Rust 2021 sin bibliotecas no estándar |
| **Rol (R)** | Ingeniero de software de sistemas y compiladores |
| **Especificación (E)** | Offset alineado al ancho de cada campo y tamaño total redondeado a la alineación mayor |
| **Output (O)** | Solo código modular y análisis de complejidad |

### 2.2 Prompt Estructurado
```text
CONTEXTO: Laboratorio de compiladores en C (C11, GCC) sobre offsets de campos de
un struct con alineación y padding. Se compila con gcc -std=c11 -Wall -Wextra
-Werror, sin warnings. Solo librerías estándar y sin main() porque el harness ya
tiene uno.

ROL: Actúa como ingeniero de software de sistemas y compiladores.

ESPECIFICACIÓN: Respeta exactamente esto de api_sesion19.h:

typedef struct Campo {
    const char *nombre;
    const char *tipo;
} Campo;

int offset_campo(const Campo *campos, int total, const char *nombre);
int tamano_struct(const Campo *campos, int total);

Reglas:
- Anchos: char=1, short=2, int=4, float=4, double=8, puntero=8.
- Cada campo se alinea al múltiplo de su propio ancho antes de darle offset.
- offset_campo regresa el offset del campo pedido, o -1 si no existe, si hay
  NULL o si un tipo es desconocido.
- tamano_struct suma los campos alineados y redondea el total al múltiplo del
  campo MÁS GRANDE del struct (no del último). total == 0 regresa 0.
  NULL o tipo desconocido regresan -1.

Ejemplos: {char,int} offset de b = 4; {char,int,short} offset de c = 8 y
tamaño 12; {double,char} tamaño 16.

OUTPUT: Solo el código de ia_sesion19_c.c, corto, y una línea con la complejidad.
```

---

## 3. Código Devuelto por el Modelo
```c
#include "api_sesion19.h"
#include <string.h>
#include <stddef.h>

static int ancho(const char *t) {
    const char *n[] = {"char", "short", "int", "float", "double", "puntero"};
    const int   w[] = { 1,      2,       4,     4,       8,        8 };
    if (!t) return -1;
    for (int i = 0; i < 6; i++) {
        if (strcmp(t, n[i]) == 0) return w[i];
    }
    return -1;
}

// redondea x hacia arriba al multiplo de a
static int redondear(int x, int a) {
    return ((x + a - 1) / a) * a;
}

int offset_campo(const Campo *campos, int total, const char *nombre) {
    if (!campos || !nombre || total < 0) return -1;
    int pos = 0;
    for (int i = 0; i < total; i++) {
        int w = ancho(campos[i].tipo);
        if (w < 0) return -1;
        pos = redondear(pos, w);
        if (campos[i].nombre && strcmp(campos[i].nombre, nombre) == 0) return pos;
        pos += w;
    }
    return -1;
}

int tamano_struct(const Campo *campos, int total) {
    if (!campos || total < 0) return -1;
    if (total == 0) return 0;
    int pos = 0, max = 1;
    for (int i = 0; i < total; i++) {
        int w = ancho(campos[i].tipo);
        if (w < 0) return -1;
        pos = redondear(pos, w) + w;
        if (w > max) max = w;   // alineacion MAYOR, no la del ultimo
    }
    return redondear(pos, max);
}

// Complejidad: offset_campo O(n), tamano_struct O(n), n = numero de campos
```

---

## 4. Fase 3: El Auditor (Resultados del Test Harness)
*Pruebas ejecutadas:* Offset con padding, struct de tres campos, tamaño con padding final, double+char, miembro inexistente y struct vacío

### Salida del harness
```text
=== AUDITORIA SESION 19 - Offsets de miembros y tamano de struct ===
Caso                               esperado:              obtenido:              resultado
[ 1] struct char e int: b               esperado: 4            obtenido: 4            OK
[ 2] struct de tres campos: c           esperado: 8            obtenido: 8            OK
[ 3] tamano total con padding           esperado: 12           obtenido: 12           OK
[ 4] double seguido de char             esperado: 16           obtenido: 16           OK
[ 5] miembro inexistente                esperado: -1           obtenido: -1           OK
[ 6] struct sin campos                  esperado: 0            obtenido: 0            OK

RESULTADO: 6/6 pruebas superadas
TODO EN VERDE. Documenta tu auditoria en auditorias/auditoria_sesion19.md
```

* [x] Caso base y límites superados sin *Crash*.
* [x] No existen accesos fuera de límites ni *Memory Leaks*.
* [x] La complejidad teórica coincide con la implementada.

### Pruebas extra que hice yo (comparando contra GCC real)
Declaré los structs de verdad en C y comparé contra `offsetof` y `sizeof`:

| Struct real | Mi resultado | GCC (`offsetof` / `sizeof`) |
| :--- | :--- | :--- |
| `{char a; double b; char c; short d;}` → b, c, d, tamaño | 8, 16, 18, 24 | 8, 16, 18, 24 |
| `{short a; char b; void *p; float f;}` → p, f, tamaño | 8, 16, 24 | 8, 16, 24 |
| `{char a; char b; char c;}` → tamaño | 3 | 3 |
| `{char a; int b;}` → tamaño | 8 | 8 |
| `{char a; long b;}` (tipo que no está en la tabla) | -1 | — |
| lista NULL | -1 | — |

### Reporte Crítico
*¿Qué caso borde o problema omitió la IA y cómo lo solucionaste?*
> En la primera versión de la IA, `tamano_struct` hacía el padding final usando el ancho del **último** campo, no el del más grande: al final tenía `return redondear(pos, w);` con la `w` del último campo del `for`. En `{char, int, short}` tuvo suerte porque daba 10 redondeado a 2 = 10, y aun así falló el caso 3, que espera 12. En `{double, char}` daba 9 redondeado a 1 = 9, cuando debía ser 16.
>
> Esta vez sí lo detectó el harness: los casos 3 y 4 salieron FALLA y me dio 4/6. Al ver el código vi que usaba la última `w` en lugar del máximo.
>
> Lo arreglé poniendo en el contrato que se redondea al campo **más grande**, no al último, con el ejemplo de `{double, char} = 16`. La segunda versión ya lleva una variable `max`. Para estar seguro de que no le atinaba de casualidad, comparé contra `sizeof` y `offsetof` de structs reales con 4 campos mezclados, y todos coinciden.
>
> Conclusión: el padding del final es lo que más se olvida, porque la suma "se ve" bien. Hay que pensar en qué pasa si haces un arreglo de ese struct.

---
**Firma digital:** ReyesDiego002
