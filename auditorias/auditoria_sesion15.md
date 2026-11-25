# Bitácora de Requerimientos y Auditoría IA — Sesión 15
**Materia:** Lenguajes y Autómatas II (SCD-1016)  
**Tema:** Bloques Anidados y Ciclo de Vida de Identificadores  
**Docente Titular:** MCC. Iván Márquez Larios (`ijmarquezl`)  
**Estudiante:** Diego Reyes Martínez  
**Matrícula:** 24531376  
**Track:** C (el harness evalúa C; la versión en Rust es opcional y no se califica con este harness)  

---

## 1. Fase 1: El Artesano (Dificultad Manual)
*Reto:* Push y pop de ámbitos con memoria dinámica  
*Describe qué problema o restricción de memoria/sintaxis identificaste al codificar a mano:*
> El push fue fácil: `malloc`, le pongo el nombre y hago que su `padre` apunte al actual. Se me olvidó incluir `stdlib.h` y el compilador me marcó `malloc` como función no declarada. Como está con `-Werror`, ni compiló hasta que lo agregué.
>
> En el pop me atoré en el orden. Primero escribí `free(actual); return actual->padre;`, y eso está mal porque después del `free` ya no puedo leer `actual`, esa memoria ya no es mía. Lo cambié para guardar el padre en una variable antes del `free`.
>
> La otra cosa fue la raíz. En el harness la raíz se crea así: `Ambito raiz = {"global", NULL};`, o sea en la pila, no con `malloc`. Si le hago `free` a eso el programa truena. Por eso, si `padre == NULL` (es la raíz), regreso NULL y no libero nada.

---

## 2. Fase 2: El Científico (Ingeniería de Requerimientos C-R-E-O)

### 2.1 Matriz de Especificación Formal
| Elemento C-R-E-O | Definición Técnica |
| :--- | :--- |
| **Contexto (C)** | C99 / GCC o Rust 2021 sin bibliotecas no estándar |
| **Rol (R)** | Ingeniero de software de sistemas y compiladores |
| **Especificación (E)** | Push con malloc enlazado al padre y pop que libera solo el actual, sin tocar la raíz |
| **Output (O)** | Solo código modular y análisis de complejidad |

### 2.2 Prompt Estructurado
```text
CONTEXTO: Laboratorio de compiladores en C (C11, GCC) sobre bloques anidados.
Cada { hace push de un ámbito y cada } hace pop. Se compila con
gcc -std=c11 -Wall -Wextra -Werror, sin warnings. Solo librerías estándar y sin
main() porque el harness ya tiene uno. La raíz la crea el harness en la pila,
no con malloc, y su padre es NULL.

ROL: Actúa como ingeniero de software de sistemas y compiladores.

ESPECIFICACIÓN: Respeta exactamente esto de api_sesion15.h:

typedef struct Ambito {
    const char *nombre;
    struct Ambito *padre;
} Ambito;

Ambito *push_ambito(Ambito *actual, const char *nombre);
Ambito *pop_ambito(Ambito *actual);

Reglas:
- push: malloc de un ámbito nuevo, nombre = nombre, padre = actual, y lo regresa.
  Si actual o nombre son NULL o falla malloc, regresa NULL.
- pop: guarda el padre, hace free del actual y regresa el padre.
  Si actual es NULL o es la raíz (padre == NULL) regresa NULL y NO hace free.
  Solo libera el ámbito que recibe.

OUTPUT: Solo el código de ia_sesion15_c.c, corto, y una línea con la complejidad.
```

---

## 3. Código Devuelto por el Modelo
```c
#include "api_sesion15.h"
#include <stdlib.h>

Ambito *push_ambito(Ambito *actual, const char *nombre) {
    if (!actual || !nombre) return NULL;
    Ambito *a = malloc(sizeof *a);
    if (!a) return NULL;
    a->nombre = nombre;
    a->padre = actual;
    return a;
}

Ambito *pop_ambito(Ambito *actual) {
    // la raiz (sin padre) no se libera porque no la creo push
    if (!actual || !actual->padre) return NULL;
    Ambito *padre = actual->padre;
    free(actual);
    return padre;
}

// Complejidad: push O(1), pop O(1)
```

---

## 4. Fase 3: El Auditor (Resultados del Test Harness)
*Pruebas ejecutadas:* Nombre del ámbito, profundidad con push, pop al padre, pop en raíz, push con NULL y profundidad tras pop

### Salida del harness
```text
=== AUDITORIA SESION 15 - Push y pop de ambitos ===
Caso                               esperado:              obtenido:              resultado
[ 1] nombre del ambito nuevo            esperado: if           obtenido: if           OK
[ 2] profundidad con dos push           esperado: 3            obtenido: 3            OK
[ 3] pop devuelve el padre              esperado: 1            obtenido: 1            OK
[ 4] pop sobre la raiz                  esperado: 1            obtenido: 1            OK
[ 5] push con padre NULL                esperado: 1            obtenido: 1            OK
[ 6] profundidad tras pop               esperado: 2            obtenido: 2            OK

RESULTADO: 6/6 pruebas superadas
TODO EN VERDE. Documenta tu auditoria en auditorias/auditoria_sesion15.md
```

* [x] Caso base y límites superados sin *Crash*.
* [x] No existen accesos fuera de límites ni *Memory Leaks*.
* [x] La complejidad teórica coincide con la implementada.

### Pruebas extra que hice yo
| Prueba | Esperado | Obtenido |
| :--- | :--- | :--- |
| `push_ambito(&raiz, NULL)` | NULL | NULL |
| `pop_ambito(NULL)` | NULL | NULL |
| Harness compilado con `-fsanitize=address` | sin fugas ni free inválido | limpio, 6/6 |

### Reporte Crítico
*¿Qué caso borde o problema omitió la IA y cómo lo solucionaste?*
> La primera versión de la IA hacía el `pop` así: guardaba el padre, hacía `free(actual)` y regresaba el padre, sin revisar si era la raíz. Para los hijos funcionaba bien, pero en el caso 4 (`pop sobre la raiz`) le hacía `free` a la raíz, que el harness creó en la pila y no con `malloc`. El programa tronaba con `free(): invalid pointer` y ni siquiera llegaba a imprimir el resultado.
>
> Lo detecté porque la celda del harness se cortó en el caso 4 con ese error. Al ver el código noté que nunca preguntaba `if (actual->padre == NULL)`.
>
> Lo arreglé poniendo en el contrato que la raíz no se creó con `malloc` y que si `padre == NULL` regrese NULL sin hacer `free`. Después lo compilé con `-fsanitize=address` para confirmar que no quedaran fugas ni frees inválidos, y salió limpio.
>
> Lo que aprendí es que solo puedes hacer `free` de lo que tú reservaste con `malloc`. Aunque el puntero se vea igual, si la memoria viene de otro lado no te toca liberarla.

---
**Firma digital:** ReyesDiego002
