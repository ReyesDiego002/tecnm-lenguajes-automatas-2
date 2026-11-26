# Bitácora de Requerimientos y Auditoría IA — Sesión 25
**Materia:** Lenguajes y Autómatas II (SCD-1016)  
**Tema:** Control de Flujo: Bucles While y For  
**Docente Titular:** MCC. Iván Márquez Larios (`ijmarquezl`)  
**Estudiante:** Diego Reyes Martínez  
**Matrícula:** 24531376  
**Track:** C (el harness evalúa C; la versión en Rust es opcional y no se califica con este harness)  

---

## 1. Fase 1: El Artesano (Dificultad Manual)
*Reto:* Esqueleto TAC de while y for con etiquetas de inicio y fin  
*Describe qué problema o restricción de memoria/sintaxis identificaste al codificar a mano:*
> Antes de programar lo dibujé en papel para entender a dónde salta cada cosa. El `IFFALSE` va al **final** (para salir del ciclo) y el `GOTO` va al **inicio** (para repetir). Al principio los tenía al revés y el ciclo nunca iba a terminar.
>
> Lo de las etiquetas lo saqué de las sesiones 21 y 23: `(*etiquetas)++` dos veces, una para el inicio y otra para el fin. Así, el segundo `while` ya empieza en L2 sin que yo haga nada especial.
>
> Como iba a repetir mucho el "escribe en `lineas[*n]` y súbele a `*n`", hice una función `agregar()` que además revisa `*n < max`. Así nada más armo el texto con `snprintf` en un `buf` y se lo paso.
>
> En el `for` me costó decidir dónde va el `i = i + 1`. Si lo pongo después del `GOTO` nunca se ejecuta, porque el `GOTO` ya brincó. Tiene que ir justo antes del `GOTO`. El caso 5 del harness revisa que esté en la línea 3.
>
> Y para los NULL, puse `if (inicio != NULL)` y `if (post != NULL)` antes de agregarlos, para que esa línea ni exista.

---

## 2. Fase 2: El Científico (Ingeniería de Requerimientos C-R-E-O)

### 2.1 Matriz de Especificación Formal
| Elemento C-R-E-O | Definición Técnica |
| :--- | :--- |
| **Contexto (C)** | C99 / GCC o Rust 2021 sin bibliotecas no estándar |
| **Rol (R)** | Ingeniero de software de sistemas y compiladores |
| **Especificación (E)** | While de 4 líneas y for de 6 (init, marca, IFFALSE, post, GOTO, fin) con 2 etiquetas únicas por ciclo; init/post NULL se omiten |
| **Output (O)** | Solo código modular y análisis de complejidad |

### 2.2 Prompt Estructurado
```text
CONTEXTO: Laboratorio de compiladores en C (C11, GCC) sobre traducir while y for
a código de tres direcciones. Se compila con gcc -std=c11 -Wall -Wextra -Werror,
sin warnings. Solo librerías estándar y sin main() porque el harness ya tiene uno.
Las funciones agregan líneas a lineas[] desde la posición *n y suben *n.

ROL: Actúa como ingeniero de software de sistemas y compiladores.

ESPECIFICACIÓN: Respeta exactamente estas firmas de api_sesion25.h:

void emitir_while(char lineas[][64], int max, int *n, const char *cond,
                  int *etiquetas);
void emitir_for(char lineas[][64], int max, int *n, const char *inicio,
                const char *cond, const char *post, int *etiquetas);

Reglas:
- Cada ciclo toma 2 etiquetas: ini = (*etiquetas)++, fin = (*etiquetas)++.
- while escribe: "L{ini}:", "IFFALSE {cond} GOTO L{fin}", "GOTO L{ini}", "L{fin}:".
- for escribe: "{inicio}", "L{ini}:", "IFFALSE {cond} GOTO L{fin}", "{post}",
  "GOTO L{ini}", "L{fin}:".
- Si inicio es NULL esa línea NO se escribe (ni vacía ni "(null)").
  Si post es NULL tampoco. El post va ANTES del GOTO.
- Nunca más de max líneas. lineas, n, cond o etiquetas NULL: no hacer nada.

Ejemplos: dos while seguidos -> el segundo empieza en "L2:".
for("j = 0","j < 3",NULL) -> 5 líneas.

OUTPUT: Solo el código de ia_sesion25_c.c, corto, y una línea con la complejidad.
```

---

## 3. Código Devuelto por el Modelo
```c
#include "api_sesion25.h"
#include <stdio.h>
#include <stddef.h>

// escribe una linea solo si cabe y avanza *n
static void linea(char l[][64], int max, int *n, const char *fmt, const char *s, int k) {
    if (*n >= max) return;
    if (s) snprintf(l[*n], 64, fmt, s, k);
    else   snprintf(l[*n], 64, fmt, k);
    (*n)++;
}

void emitir_while(char lineas[][64], int max, int *n, const char *cond, int *etiquetas) {
    if (!lineas || !n || !cond || !etiquetas) return;
    int ini = (*etiquetas)++, fin = (*etiquetas)++;
    linea(lineas, max, n, "L%d:", NULL, ini);
    linea(lineas, max, n, "IFFALSE %s GOTO L%d", cond, fin);
    linea(lineas, max, n, "GOTO L%d", NULL, ini);
    linea(lineas, max, n, "L%d:", NULL, fin);
}

void emitir_for(char lineas[][64], int max, int *n, const char *inicio, const char *cond, const char *post, int *etiquetas) {
    if (!lineas || !n || !cond || !etiquetas) return;
    int ini = (*etiquetas)++, fin = (*etiquetas)++;
    if (inicio) linea(lineas, max, n, "%s", inicio, 0);   // NULL = no se emite
    linea(lineas, max, n, "L%d:", NULL, ini);
    linea(lineas, max, n, "IFFALSE %s GOTO L%d", cond, fin);
    if (post) linea(lineas, max, n, "%s", post, 0);       // actualizacion antes del GOTO
    linea(lineas, max, n, "GOTO L%d", NULL, ini);
    linea(lineas, max, n, "L%d:", NULL, fin);
}

// Complejidad: O(1) por ciclo (4 o 6 lineas fijas)
```

---

## 4. Fase 3: El Auditor (Resultados del Test Harness)
*Pruebas ejecutadas:* Marca de inicio y condición del while, dos ciclos sin colisión, inicialización y actualización del for, for sin post

### Salida del harness
```text
=== AUDITORIA SESION 25 - Esqueleto TAC de while y for ===
Caso                               esperado:              obtenido:              resultado
[ 1] while: marca de inicio             esperado: L0:          obtenido: L0:          OK
[ 2] while: condicion
     esperado: IFFALSE a < b GOTO L1
     obtenido: IFFALSE a < b GOTO L1
     -> OK
[ 3] dos ciclos sin colision            esperado: L2:          obtenido: L2:          OK
[ 4] for: inicializacion                esperado: i = 0        obtenido: i = 0        OK
[ 5] for: actualizacion                 esperado: i = i + 1    obtenido: i = i + 1    OK
[ 6] for sin post (NULL)                esperado: 5            obtenido: 5            OK

RESULTADO: 6/6 pruebas superadas
TODO EN VERDE. Documenta tu auditoria en auditorias/auditoria_sesion25.md
```

* [x] Caso base y límites superados sin *Crash*.
* [x] No existen accesos fuera de límites ni *Memory Leaks*.
* [x] La complejidad teórica coincide con la implementada.

### Pruebas extra que hice yo (imprimiendo el ciclo completo)
**Dos while seguidos** (`a < b` y `c < d`):
```text
0: L0:
1: IFFALSE a < b GOTO L1
2: GOTO L0
3: L1:
4: L2:
5: IFFALSE c < d GOTO L3
6: GOTO L2
7: L3:
```
**for completo** (`i = 0; i < 10; i = i + 1`):
```text
0: i = 0
1: L0:
2: IFFALSE i < 10 GOTO L1
3: i = i + 1
4: GOTO L0
5: L1:
```

| Prueba | Esperado | Obtenido |
| :--- | :--- | :--- |
| for con `inicio` NULL | 5 líneas, empieza en `L0:` | 5 líneas, empieza en `L0:` |
| for completo con `max = 3` | se para en 3 sin desbordar | 3 líneas |
| while con `cond` NULL | no escribe nada | 0 líneas |
| Etiquetas después de 2 while | 4 | 4 |

Compilado con `-fsanitize=address`, sin errores de memoria.

### Reporte Crítico
*¿Qué caso borde o problema omitió la IA y cómo lo solucionaste?*
> En la primera versión de la IA, `emitir_for` revisaba `if (post)` para no escribir la actualización, pero la inicialización la escribía siempre: `snprintf(lineas[(*n)++], 64, "%s", inicio);`. El harness solo prueba `post = NULL` (caso 6), nunca `inicio = NULL`, así que salía 6/6.
>
> La sección 1 de la práctica dice que un NULL en `inicio` **o** en `post` significa que no se emite esa instrucción. Hice la prueba con `emitir_for(..., NULL, "k < 5", "k = k + 1", ...)`, como un `for (; k < 5; k++)`, y la versión de la IA escribía una línea `(null)` al principio, porque así imprime glibc un `%s` con NULL. Con eso daba 6 líneas en lugar de 5, y una de ellas es basura que no es TAC válido.
>
> Lo arreglé poniendo en el contrato que si `inicio` es NULL esa línea no se escribe, "ni vacía ni (null)". La segunda versión ya trae `if (inicio)` antes de esa línea.
>
> Conclusión: si el harness prueba que un parámetro puede ser NULL, hay que revisar si los demás también pueden serlo. El contrato lo decía para los dos.

---
**Firma digital:** ReyesDiego002
