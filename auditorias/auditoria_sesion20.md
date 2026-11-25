# Bitácora de Requerimientos y Auditoría IA — Sesión 20
**Materia:** Lenguajes y Autómatas II (SCD-1016)  
**Tema:** Hito 2: Analizador Semántico Completo y Scopes  
**Docente Titular:** MCC. Iván Márquez Larios (`ijmarquezl`)  
**Estudiante:** Diego Reyes Martínez  
**Matrícula:** 24531376  
**Track:** C (el harness evalúa C; la versión en Rust es opcional y no se califica con este harness)  

---

## 1. Fase 1: El Artesano (Dificultad Manual)
*Reto:* Resolver nombres con profundidad y contar símbolos visibles con shadowing  
*Describe qué problema o restricción de memoria/sintaxis identificaste al codificar a mano:*
> `profundidad_de` salió rápido porque es casi igual al `buscar` de la sesión 13. La única diferencia es que en vez de regresar el nombre del ámbito llevo un contador `nivel` que sube cada vez que me voy al padre. Lo que sí cuidé fue revisar NULL al principio para regresar -2, porque si no, el `actual->total` truena.
>
> `contar_visibles` fue lo difícil. Mi primera idea fue sumar el `total` de todos los ámbitos, pero eso daba 4 en el caso 5, porque `y` está en los dos niveles. Luego pensé en guardar los nombres que ya conté en un arreglo, pero ¿de qué tamaño? Si hay muchos niveles no sé cuántos nombres son.
>
> Al final lo hice sin arreglo extra: por cada nombre, reviso dos cosas. Si ya salió antes en el mismo ámbito, o si está en algún ámbito más adentro (entre `actual` y donde estoy), es que ya lo conté y lo salto. Si no, cuento uno. Es más lento, pero no me salgo de la memoria.
>
> Mi primera versión de esto calculaba la profundidad y luego caminaba hasta el "dueño" del nombre, y estaba bien pero muy enredada. La simplifiqué con dos `for` que se leen mejor.

---

## 2. Fase 2: El Científico (Ingeniería de Requerimientos C-R-E-O)

### 2.1 Matriz de Especificación Formal
| Elemento C-R-E-O | Definición Técnica |
| :--- | :--- |
| **Contexto (C)** | C99 / GCC o Rust 2021 sin bibliotecas no estándar |
| **Rol (R)** | Ingeniero de software de sistemas y compiladores |
| **Especificación (E)** | Profundidad del ámbito más cercano (-1 / -2) y conteo de nombres distintos visibles sin buffer fijo |
| **Output (O)** | Solo código modular y análisis de complejidad |

### 2.2 Prompt Estructurado
```text
CONTEXTO: Hito 2 de un laboratorio de compiladores en C (C11, GCC): resolución de
nombres con ámbitos anidados. Se compila con gcc -std=c11 -Wall -Wextra -Werror,
sin warnings. Solo librerías estándar y sin main() porque el harness ya tiene uno.

ROL: Actúa como ingeniero de software de sistemas y compiladores.

ESPECIFICACIÓN: Respeta exactamente esto de api_sesion20.h:

#define AMBITO_MAX 8
typedef struct Ambito {
    const char *nombre;
    const char *simbolos[AMBITO_MAX];
    int total;
    struct Ambito *padre;
} Ambito;

int profundidad_de(const Ambito *actual, const char *nombre);
int contar_visibles(const Ambito *actual);

Reglas:
- profundidad_de: nivel 0 es actual, sube por padre. Regresa el nivel del
  ámbito más cercano que tenga el nombre, -1 si no existe, -2 si actual o
  nombre son NULL.
- contar_visibles: cuenta nombres DISTINTOS desde actual hasta la raíz
  (un nombre en varios niveles cuenta una vez). -2 si actual es NULL.
  No uses un arreglo fijo para los nombres vistos (puede haber muchos niveles);
  revisa duplicados recorriendo la cadena.

Ejemplos: global {x,y} + bloque {y,z} = 3 visibles; "x" en hijo y padre = 0.

OUTPUT: Solo el código de ia_sesion20_c.c, corto, y una línea con la complejidad.
```

---

## 3. Código Devuelto por el Modelo
```c
#include "api_sesion20.h"
#include <string.h>
#include <stddef.h>

static int esta_en(const Ambito *a, const char *n, int hasta) {
    for (int i = 0; i < hasta; i++) {
        if (strcmp(a->simbolos[i], n) == 0) return 1;
    }
    return 0;
}

int profundidad_de(const Ambito *actual, const char *nombre) {
    if (!actual || !nombre) return -2;
    int nivel = 0;
    for (const Ambito *a = actual; a != NULL; a = a->padre, nivel++) {
        if (esta_en(a, nombre, a->total)) return nivel;
    }
    return -1;
}

int contar_visibles(const Ambito *actual) {
    if (!actual) return -2;
    int cuenta = 0;
    for (const Ambito *a = actual; a != NULL; a = a->padre) {
        for (int i = 0; i < a->total; i++) {
            const char *s = a->simbolos[i];
            int ya = esta_en(a, s, i);              // repetido en el mismo nivel
            for (const Ambito *b = actual; b != a && !ya; b = b->padre) {
                ya = esta_en(b, s, b->total);       // ya visto en un nivel mas interno
            }
            if (!ya) cuenta++;
        }
    }
    return cuenta;
}

// Complejidad: profundidad_de O(h*n); contar_visibles O((h*n)^2), h = niveles, n = simbolos por nivel
```

---

## 4. Fase 3: El Auditor (Resultados del Test Harness)
*Pruebas ejecutadas:* Nombre local, dos niveles arriba, shadowing, nombre no visible, visibles sin repetir y ámbito NULL

### Salida del harness
```text
=== AUDITORIA SESION 20 - Resolucion de nombres y visibilidad ===
Caso                               esperado:              obtenido:              resultado
[ 1] nombre en el ambito actual         esperado: 0            obtenido: 0            OK
[ 2] nombre dos niveles arriba          esperado: 2            obtenido: 2            OK
[ 3] shadowing gana el interno          esperado: 0            obtenido: 0            OK
[ 4] nombre no visible                  esperado: -1           obtenido: -1           OK
[ 5] visibles sin repetir               esperado: 3            obtenido: 3            OK
[ 6] ambito NULL                        esperado: -2           obtenido: -2           OK

RESULTADO: 6/6 pruebas superadas
TODO EN VERDE. Documenta tu auditoria en auditorias/auditoria_sesion20.md
```

* [x] Caso base y límites superados sin *Crash*.
* [x] No existen accesos fuera de límites ni *Memory Leaks*.
* [x] La complejidad teórica coincide con la implementada.

### Pruebas extra que hice yo
| Prueba | Esperado | Obtenido |
| :--- | :--- | :--- |
| 3 niveles con 8 nombres distintos cada uno | 24 | 24 |
| 3 niveles con los mismos `x, y, z` en cada uno | 3 | 3 |
| Ámbito sin símbolos | 0 | 0 |
| `"x"` dos veces en el mismo ámbito | 1 | 1 |
| `contar_visibles(NULL)` | -2 | -2 |
| `profundidad_de(NULL, NULL)` | -2 | -2 |

Todas compiladas con `-fsanitize=address` y sin errores de memoria.

### Reporte Crítico
*¿Qué caso borde o problema omitió la IA y cómo lo solucionaste?*
> En la primera versión de la IA, `contar_visibles` guardaba los nombres ya contados en un arreglo local `const char *vistos[AMBITO_MAX];`, o sea de 8 lugares. En el harness solo hay 4 nombres en total, así que cabía y salía 6/6. Pero con 3 niveles de 8 nombres distintos son 24, y el arreglo `vistos` se desbordaba desde el noveno nombre, escribiendo en la pila del programa.
>
> Lo detecté porque en la Fase 1 yo mismo me pregunté de qué tamaño tenía que ser ese arreglo. Hice una prueba con 3 niveles de 8 nombres, compilé con `-fsanitize=address` y salió `stack-buffer-overflow`.
>
> Lo arreglé poniendo en el contrato que no se sabe cuántos niveles hay y que no use un arreglo fijo, que revise los duplicados recorriendo la cadena. La segunda versión ya no tiene buffer extra: por cada nombre revisa si ya estaba antes en su mismo ámbito o en un ámbito más interno. Es más lenta (O((h·n)²)), pero no tiene límite.
>
> Conclusión: `AMBITO_MAX` es el límite de **un** ámbito, no de toda la cadena. La IA confundió los dos, y el harness no lo nota porque usa cadenas chiquitas.

---
**Firma digital:** ReyesDiego002
