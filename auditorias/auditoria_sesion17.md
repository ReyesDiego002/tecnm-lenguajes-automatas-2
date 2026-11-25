# Bitácora de Requerimientos y Auditoría IA — Sesión 17
**Materia:** Lenguajes y Autómatas II (SCD-1016)  
**Tema:** Comprobación Semántica de Retornos y Rutas de Control  
**Docente Titular:** MCC. Iván Márquez Larios (`ijmarquezl`)  
**Estudiante:** Diego Reyes Martínez  
**Matrícula:** 24531376  
**Track:** C (el harness evalúa C; la versión en Rust es opcional y no se califica con este harness)  

---

## 1. Fase 1: El Artesano (Dificultad Manual)
*Reto:* Validar el tipo del return y que todas las rutas terminen en return  
*Describe qué problema o restricción de memoria/sintaxis identificaste al codificar a mano:*
> Lo primero que me atoró fue que el contrato no dice qué tan ancho es cada tipo. Me acordé de la sesión 08 (char < int < float) y le agregué double arriba. Hice una función `rango()` que regresa un número por tipo, y así solo comparo números.
>
> El `void` lo tuve que tratar aparte, porque no es "más chico" que int, simplemente no es un valor. Si uno de los dos es void, los dos tienen que ser void. Si no, es error de tipo (1).
>
> Lo que más me costó fue el orden de los `if`. Mi primera versión revisaba las rutas primero y luego el tipo. Así, `int` devolviendo `double` con una ruta faltante me daba 2, y el criterio dice que el tipo va primero. Cambié el orden: primero los -1, luego el tipo y al final las rutas.
>
> También agregué el caso de "más returns que rutas" como -1, porque no tiene sentido tener 3 returns si solo hay 1 camino.

---

## 2. Fase 2: El Científico (Ingeniería de Requerimientos C-R-E-O)

### 2.1 Matriz de Especificación Formal
| Elemento C-R-E-O | Definición Técnica |
| :--- | :--- |
| **Contexto (C)** | C99 / GCC o Rust 2021 sin bibliotecas no estándar |
| **Rol (R)** | Ingeniero de software de sistemas y compiladores |
| **Especificación (E)** | Diagnóstico único: -1 inválido, 1 tipo incompatible, 2 ruta sin return, 0 válido (tipo antes que rutas) |
| **Output (O)** | Solo código modular y análisis de complejidad |

### 2.2 Prompt Estructurado
```text
CONTEXTO: Laboratorio de compiladores en C (C11, GCC) sobre comprobación semántica
de returns y rutas de control. Se compila con gcc -std=c11 -Wall -Wextra -Werror,
sin warnings. Solo librerías estándar y sin main() porque el harness ya tiene uno.

ROL: Actúa como ingeniero de software de sistemas y compiladores.

ESPECIFICACIÓN: Respeta exactamente esta firma de api_sesion17.h:

int diagnosticar_retorno(const char *tipo_funcion, const char *tipo_expresion,
                         int rutas_totales, int rutas_con_return);

Regresa un solo diagnóstico:
- -1 si algún tipo es NULL o desconocido, rutas_totales <= 0,
  rutas_con_return < 0 o rutas_con_return > rutas_totales.
- 1 si el tipo es incompatible: void con no-void (o al revés), o la expresión
  es más ancha que la función. Jerarquía: char < int < float < double.
- 2 si el tipo está bien pero rutas_con_return < rutas_totales.
- 0 si todo está bien.
IMPORTANTE: revisa el tipo ANTES que las rutas. ("int","double",2,1) debe dar 1.

Ejemplos: ("int","int",2,2)=0, ("void","int",1,1)=1, ("int","int",2,1)=2,
("double","int",1,1)=0, ("int","double",1,1)=1, ("int","int",0,0)=-1.

OUTPUT: Solo el código de ia_sesion17_c.c, corto, y una línea con la complejidad.
```

---

## 3. Código Devuelto por el Modelo
```c
#include "api_sesion17.h"
#include <string.h>
#include <stddef.h>

// rango: void=0, char=1, int=2, float=3, double=4, desconocido=-1
static int rango(const char *t) {
    const char *tipos[] = {"void", "char", "int", "float", "double"};
    for (int i = 0; i < 5; i++) {
        if (strcmp(t, tipos[i]) == 0) return i;
    }
    return -1;
}

int diagnosticar_retorno(const char *tipo_funcion, const char *tipo_expresion, int rutas_totales, int rutas_con_return) {
    if (!tipo_funcion || !tipo_expresion) return -1;
    if (rutas_totales <= 0 || rutas_con_return < 0 || rutas_con_return > rutas_totales) return -1;

    int rf = rango(tipo_funcion), re = rango(tipo_expresion);
    if (rf < 0 || re < 0) return -1;

    // el tipo se revisa ANTES que las rutas
    if (rf == 0 || re == 0) return (rf == re) ? 0 : 1;
    if (re > rf) return 1;                         // se perderia informacion

    if (rutas_con_return < rutas_totales) return 2; // if sin else
    return 0;
}

// Complejidad: O(1)
```

---

## 4. Fase 3: El Auditor (Resultados del Test Harness)
*Pruebas ejecutadas:* Retorno válido, void con valor, ruta sin return, ensanchamiento, estrechamiento y cero rutas

### Salida del harness
```text
=== AUDITORIA SESION 17 - Validacion de retornos y rutas ===
Caso                               esperado:              obtenido:              resultado
[ 1] int con int y dos rutas            esperado: 0            obtenido: 0            OK
[ 2] void devuelve valor                esperado: 1            obtenido: 1            OK
[ 3] ruta sin retorno                   esperado: 2            obtenido: 2            OK
[ 4] double acepta int                  esperado: 0            obtenido: 0            OK
[ 5] int no acepta double               esperado: 1            obtenido: 1            OK
[ 6] cero rutas de ejecucion            esperado: -1           obtenido: -1           OK

RESULTADO: 6/6 pruebas superadas
TODO EN VERDE. Documenta tu auditoria en auditorias/auditoria_sesion17.md
```

* [x] Caso base y límites superados sin *Crash*.
* [x] No existen accesos fuera de límites ni *Memory Leaks*.
* [x] La complejidad teórica coincide con la implementada.

### Pruebas extra que hice yo
| Prueba | Esperado | Obtenido |
| :--- | :--- | :--- |
| `("int", "double", 2, 1)` tipo mal Y ruta faltante | 1 | 1 |
| `("void", "void", 2, 1)` | 0 | 0 |
| `("int", "void", 1, 1)` | 1 | 1 |
| `("int", "int", 1, 3)` más returns que rutas | -1 | -1 |
| `(NULL, "int", 1, 1)` | -1 | -1 |
| `("int", "string", 1, 1)` tipo desconocido | -1 | -1 |
| `("float", "char", 3, 3)` | 0 | 0 |

### Reporte Crítico
*¿Qué caso borde o problema omitió la IA y cómo lo solucionaste?*
> En la primera versión que me dio la IA (cuando no le dije nada del orden), revisaba primero `if (rutas_con_return < rutas_totales) return 2;` y luego el tipo. En el harness eso no se nota, porque el caso 5 (`int no acepta double`) tiene todas las rutas completas `(1, 1)`, así que llegaba al tipo y salía 1. Daba 6/6.
>
> Pero el primer criterio de aceptación dice que el tipo va antes que las rutas. Hice la prueba con `("int", "double", 2, 1)` y la IA me daba **2** cuando tenía que ser **1**. O sea, el compilador le diría al programador "te falta un return" cuando el error real es que devuelve un double en una función int.
>
> Lo arreglé poniendo en el contrato, en mayúsculas, que el tipo se revisa antes que las rutas, con ese mismo ejemplo. La segunda versión ya tiene el orden correcto.
>
> Conclusión: cuando hay más de un error a la vez, el orden de los `if` decide cuál se reporta, y el harness no prueba casos con dos errores juntos.

---
**Firma digital:** ReyesDiego002
