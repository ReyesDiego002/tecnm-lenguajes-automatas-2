# Bitácora de Requerimientos y Auditoría IA — Sesión 12
**Materia:** Lenguajes y Autómatas II (SCD-1016)  
**Tema:** Atributos de Símbolos y Cálculo de Memoria  
**Docente Titular:** MCC. Iván Márquez Larios (`ijmarquezl`)  
**Estudiante:** Diego Reyes Martínez  
**Matrícula:** 24531376  
**Track:** C (el harness evalúa C; la versión en Rust es opcional y no se califica con este harness)  

---

## 1. Fase 1: El Artesano (Dificultad Manual)
*Reto:* Ancho de tipos y offsets alineados en el marco de pila  
*Describe qué problema o restricción de memoria/sintaxis identificaste al codificar a mano:*
> `ancho_de_tipo` fue lo fácil: comparar el texto con `strcmp` y regresar el número de la tabla. Al principio el TODO solo decía char, short, int y double, pero el contrato también pide `float` y `puntero`, así que los agregué.
>
> Donde me costó fue en la alineación. Mi primera idea era hacer `offset = cursor` y ya, pero con un `char` en 0 y luego un `int`, el int quedaba en el offset 1, y eso está mal porque un int tiene que empezar en múltiplo de 4. Entonces lo que hice fue: si `cursor % ancho` no es 0, le sumo lo que le falta para llegar al siguiente múltiplo. Por ejemplo, con cursor 1 y ancho 4: `1 + (4 - 1) = 4`.
>
> También me confundí con qué regresar. El harness en el caso 3 espera 4 para un int en cursor 0, o sea que no regreso el offset, regreso dónde empieza la **siguiente** variable (`offset + ancho`). El offset se guarda en el struct.
>
> Y puse las validaciones antes de tocar el registro, para que si viene NULL o un tipo raro no escriba nada.

---

## 2. Fase 2: El Científico (Ingeniería de Requerimientos C-R-E-O)

### 2.1 Matriz de Especificación Formal
| Elemento C-R-E-O | Definición Técnica |
| :--- | :--- |
| **Contexto (C)** | C99 / GCC o Rust 2021 sin bibliotecas no estándar |
| **Rol (R)** | Ingeniero de software de sistemas y compiladores |
| **Especificación (E)** | Tabla de anchos y offset alineado al múltiplo del ancho del tipo |
| **Output (O)** | Solo código modular y análisis de complejidad |

### 2.2 Prompt Estructurado
```text
CONTEXTO: Estoy haciendo un laboratorio de compiladores en C (C11, GCC) donde se
calcula el offset de cada variable en el marco de pila respetando la alineación.
Se compila con gcc -std=c11 -Wall -Wextra -Werror, así que no puede tener warnings.
Solo librerías estándar. Mi archivo se compila con un harness que ya trae main(),
por eso NO pongas main().

ROL: Actúa como ingeniero de software de sistemas y compiladores.

ESPECIFICACIÓN: Respeta exactamente esto de api_sesion12.h:

typedef struct Simbolo {
    const char *nombre;
    const char *tipo;
    int offset;
} Simbolo;

int ancho_de_tipo(const char *tipo);
int asignar_offset(Simbolo *simbolo, int cursor);

Reglas:
- ancho_de_tipo: char=1, short=2, int=4, float=4, double=8, puntero=8.
  El tipo se llama "puntero" (en español). Cualquier otro o NULL regresa -1.
- asignar_offset: sube el cursor al siguiente múltiplo del ancho (nunca baja),
  guarda ese offset en simbolo->offset y regresa offset + ancho.
- Si simbolo es NULL, cursor < 0 o el tipo no existe, regresa -1 y no escribe
  en el registro.
- Ejemplos: int en cursor 0 regresa 4; char en 0 y luego int deja el int en
  offset 4; double desde cursor 4 queda en offset 8.

OUTPUT: Solo el código de ia_sesion12_c.c y al final una línea con la complejidad.
```

---

## 3. Código Devuelto por el Modelo
```c
#include "api_sesion12.h"
#include <string.h>
#include <stddef.h>

typedef struct {
    const char *nombre;
    int ancho;
} InfoTipo;

static const InfoTipo TIPOS[] = {
    { "char",    1 },
    { "short",   2 },
    { "int",     4 },
    { "float",   4 },
    { "double",  8 },
    { "puntero", 8 },
};

int ancho_de_tipo(const char *tipo) {
    if (tipo == NULL) return -1;
    for (size_t i = 0; i < sizeof(TIPOS) / sizeof(TIPOS[0]); i++) {
        if (strcmp(tipo, TIPOS[i].nombre) == 0) {
            return TIPOS[i].ancho;
        }
    }
    return -1;
}

int asignar_offset(Simbolo *simbolo, int cursor) {
    if (simbolo == NULL || cursor < 0) return -1;

    int ancho = ancho_de_tipo(simbolo->tipo);
    if (ancho <= 0) return -1;

    /* redondeo hacia arriba al siguiente multiplo del ancho */
    int offset = ((cursor + ancho - 1) / ancho) * ancho;

    simbolo->offset = offset;
    return offset + ancho;
}

/* Complejidad: ancho_de_tipo O(1) (tabla fija de 6 tipos); asignar_offset O(1). */
```

---

## 4. Fase 3: El Auditor (Resultados del Test Harness)
*Pruebas ejecutadas:* Anchos de int y double, alineación de char→int, double desde cursor 4 y registro NULL

### Salida del harness
```text
=== AUDITORIA SESION 12 - Registro de simbolo y offsets alineados ===
Caso                               esperado:              obtenido:              resultado
[ 1] ancho_de_tipo("int")               esperado: 4            obtenido: 4            OK
[ 2] ancho_de_tipo("double")            esperado: 8            obtenido: 8            OK
[ 3] int en cursor 0                    esperado: 4            obtenido: 4            OK
[ 4] char e int: offset de b            esperado: 4            obtenido: 4            OK
[ 5] double desde cursor 4              esperado: 8            obtenido: 8            OK
[ 6] registro NULL                      esperado: -1           obtenido: -1           OK

RESULTADO: 6/6 pruebas superadas
TODO EN VERDE. Documenta tu auditoria en auditorias/auditoria_sesion12.md
```

* [x] Caso base y límites superados sin *Crash*.
* [x] No existen accesos fuera de límites ni *Memory Leaks*.
* [x] La complejidad teórica coincide con la implementada.

> Nota: con el esqueleto original todo regresaba -1, así que solo pasaba el caso 6 y los demás daban FALLA.

### Pruebas extra que hice yo
| Prueba | Esperado | Obtenido |
| :--- | :--- | :--- |
| `ancho_de_tipo("puntero")` | 8 | 8 |
| `ancho_de_tipo("long")` (no existe) | -1 | -1 |
| int desde cursor 4 (ya alineado) | offset 4, regresa 8 | offset 4, regresa 8 |
| cursor negativo (-3) | -1 y no toca offset | -1, offset sigue igual |
| tipo NULL | -1 | -1 |
| Secuencia char, short, puntero, char, float desde 0 | offsets 0, 2, 8, 16, 20 y total 24 | 0, 2, 8, 16, 20 y 24 |

### Reporte Crítico
*¿Qué caso borde o problema omitió la IA y cómo lo solucionaste?*
> En la primera versión que me dio la IA (cuando solo le pasé las firmas y le dije "tipos de C") la tabla tenía `"char*"` y `"pointer"` en vez de `"puntero"`, y tampoco tenía `"short"`. El harness no lo detecta porque solo prueba `int`, `double` y `char`, así que salía 6/6 en verde. Pero `ancho_de_tipo("puntero")` daba -1, y cualquier variable de tipo puntero no se podía acomodar en la pila.
>
> Me di cuenta al comparar su tabla contra la que viene en los criterios de aceptación (`char=1, short=2, int=4, float=4, double=8, puntero=8`). Luego hice una prueba con los 6 tipos y vi que dos daban -1.
>
> Lo arreglé poniendo en el contrato la tabla completa y diciéndole que el nombre es literalmente `"puntero"`, en español. También le pedí que si hay error no escriba en el registro. La segunda versión ya trae la tabla bien.
>
> Lo que me llevo es que hay que leer los criterios de aceptación y no solo los casos del harness, porque el harness no prueba todos los tipos.

---
**Firma digital:** ReyesDiego002
