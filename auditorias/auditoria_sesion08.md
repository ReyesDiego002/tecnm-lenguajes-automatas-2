# Bitácora de Requerimientos y Auditoría IA — Sesión 08
**Materia:** Lenguajes y Autómatas II (SCD-1016)  
**Tema:** Inferencia y Promoción de Tipos  
**Docente Titular:** MCC. Iván Márquez Larios (`ijmarquezl`)  
**Estudiante:** Diego Reyes Martínez  
**Matrícula:** 24531376  
**Track:** C (el harness evalúa C; la versión en Rust es opcional y no se califica con este harness)  

---

## 1. Fase 1: El Artesano (Dificultad Manual)
*Reto:* Reglas de casteo implícito  
*Describe qué problema o restricción de memoria/sintaxis identificaste al codificar a mano:*
> Lo primero que pensé fue usar directamente los valores del `enum` para comparar, porque `CHAR=1 < INT=2 < FLOAT=3` ya sigue la jerarquía. El problema es que `TIPO_ERROR` vale 0, entonces si solo regreso el mayor, `promover(ERROR, INT)` daba `INT` y el error se "perdía". Tuve que revisar ERROR **antes** de comparar para que siempre se propague.
>
> En `hay_perdida` me pasó algo parecido: como ERROR es 0, `hay_perdida(ERROR, FLOAT)` daba 1 aunque esa conversión ni siquiera es válida. Le agregué la misma revisión al inicio.
>
> En `nombre_tipo` usé un `switch` con todos los casos del enum. Como compila con `-Wall -Wextra -Werror`, si me faltaba un caso o un `return` al final, el compilador lo marcaba como error, así que dejé un `return "desconocido"` después del `switch`.

---

## 2. Fase 2: El Científico (Ingeniería de Requerimientos C-R-E-O)

### 2.1 Matriz de Especificación Formal
| Elemento C-R-E-O | Definición Técnica |
| :--- | :--- |
| **Contexto (C)** | C99 / GCC o Rust 2021 sin bibliotecas no estándar |
| **Rol (R)** | Ingeniero de software de sistemas y compiladores |
| **Especificación (E)** | Promoción entera y ensanchamiento |
| **Output (O)** | Solo código modular y análisis de complejidad |

### 2.2 Prompt Estructurado
```text
CONTEXTO: Estoy en un laboratorio de compiladores en C (C11, GCC) sin bibliotecas
no estándar. El código se compila con: gcc -std=c11 -Wall -Wextra -Werror, así que
no puede tener advertencias. Mi archivo se compila junto con un harness que ya
trae main(), por eso NO debe tener main().

ROL: Actúa como ingeniero de software de sistemas y compiladores.

ESPECIFICACIÓN: Implementa estas 3 funciones respetando EXACTAMENTE esta cabecera:

typedef enum { TIPO_ERROR = 0, TIPO_CHAR = 1, TIPO_INT = 2, TIPO_FLOAT = 3 } Tipo;
Tipo promover(Tipo a, Tipo b);
int hay_perdida(Tipo destino, Tipo origen);
const char *nombre_tipo(Tipo t);

Reglas:
- Jerarquía: CHAR < INT < FLOAT.
- promover devuelve el tipo más ancho de los dos; si alguno es TIPO_ERROR
  devuelve TIPO_ERROR.
- hay_perdida devuelve 1 cuando el destino es más angosto que el origen, 0 si no.
- nombre_tipo devuelve "char", "int", "float" o "error".
- Incluye "api_sesion08.h" en lugar de redefinir el enum.

OUTPUT: Solo el código del archivo ia_sesion08_c.c y al final una línea con la
complejidad de cada función.
```

---

## 3. Código Devuelto por el Modelo
```c
#include "api_sesion08.h"

/* rango de cada tipo: CHAR=1, INT=2, FLOAT=3, ERROR=0 (invalido) */
static int rango(Tipo t) {
    if (t == TIPO_CHAR)  return 1;
    if (t == TIPO_INT)   return 2;
    if (t == TIPO_FLOAT) return 3;
    return 0;
}

Tipo promover(Tipo a, Tipo b) {
    if (rango(a) == 0 || rango(b) == 0) {
        return TIPO_ERROR; /* el centinela ERROR se propaga */
    }
    return (rango(a) >= rango(b)) ? a : b;
}

int hay_perdida(Tipo destino, Tipo origen) {
    if (rango(destino) == 0 || rango(origen) == 0) {
        return 0; /* conversion invalida, no se reporta perdida */
    }
    return rango(destino) < rango(origen);
}

const char *nombre_tipo(Tipo t) {
    switch (t) {
        case TIPO_CHAR:  return "char";
        case TIPO_INT:   return "int";
        case TIPO_FLOAT: return "float";
        case TIPO_ERROR: return "error";
    }
    return "desconocido";
}

/* Complejidad: promover O(1), hay_perdida O(1), nombre_tipo O(1). Memoria O(1). */
```

---

## 4. Fase 3: El Auditor (Resultados del Test Harness)
*Pruebas ejecutadas:* Mezcla de tipos enteros y flotantes

### Salida del harness
```text
=== AUDITORIA SESION 08 - Reglas de promocion y perdida ===
Caso                               esperado:              obtenido:              resultado
[ 1] promover(CHAR,INT)                 esperado: int          obtenido: int          OK
[ 2] promover(INT,FLOAT)                esperado: float        obtenido: float        OK
[ 3] promover(FLOAT,FLOAT)              esperado: float        obtenido: float        OK
[ 4] promover(ERROR,INT)                esperado: error        obtenido: error        OK
[ 5] hay_perdida(CHAR,INT)              esperado: 1            obtenido: 1            OK
[ 6] hay_perdida(INT,CHAR)              esperado: 0            obtenido: 0            OK

RESULTADO: 6/6 pruebas superadas
TODO EN VERDE. Documenta tu auditoria en auditorias/auditoria_sesion08.md
```

* [x] Caso base y límites superados sin *Crash*.
* [x] No existen accesos fuera de límites ni *Memory Leaks*.
* [x] La complejidad teórica coincide con la implementada.

### Reporte Crítico
*¿Qué caso borde o problema omitió la IA y cómo lo solucionaste?*
> La primera propuesta de la IA en `hay_perdida` era solo `return destino < origen;`. Con eso los casos 5 y 6 del harness salen OK, pero como `TIPO_ERROR` vale 0, `hay_perdida(ERROR, FLOAT)` daba **1**, es decir, reportaba "pérdida" en una conversión que ni es válida. El harness no prueba ese caso, así que no salía FALLA.
>
> Lo detecté leyendo el código y comparándolo con mi versión del Artesano, donde ya me había pasado lo mismo. Para confirmarlo hice una prueba extra a mano con `printf("%d\n", hay_perdida(TIPO_ERROR, TIPO_FLOAT));`.
>
> Lo solucioné pidiéndole que revisara ERROR antes de comparar. Por eso la versión final usa la función `rango()`, que regresa 0 para ERROR, y las dos funciones descartan ese caso primero. Así ERROR nunca se "gana" en `promover` y nunca cuenta como tipo angosto en `hay_perdida`.
>
> Conclusión: que el harness salga en verde no quiere decir que todo esté bien. Solo comprueba los casos que tiene, y los casos con ERROR los tuve que revisar yo.

---
**Firma digital:** ReyesDiego002
