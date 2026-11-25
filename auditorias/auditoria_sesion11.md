# Bitácora de Requerimientos y Auditoría IA — Sesión 11
**Materia:** Lenguajes y Autómatas II (SCD-1016)  
**Tema:** Arquitectura de la Tabla de Símbolos  
**Docente Titular:** MCC. Iván Márquez Larios (`ijmarquezl`)  
**Estudiante:** Diego Reyes Martínez  
**Matrícula:** 24531376  
**Track:** C (el harness evalúa C; la versión en Rust es opcional y no se califica con este harness)  

---

## 1. Fase 1: El Artesano (Dificultad Manual)
*Reto:* Tabla hash con djb2 y encadenamiento de colisiones  
*Describe qué problema o restricción de memoria/sintaxis identificaste al codificar a mano:*
> El hash djb2 salió rápido, porque es solo empezar en 5381 y en cada letra hacer `h * 33 + letra`. Lo que sí me hizo dudar fue si la letra tenía que ser `char` o `unsigned char`. Al final le puse `(unsigned char)` porque si alguna letra tiene acento el `char` puede salir negativo y el hash cambia.
>
> Donde más me atoré fue en `insertar_simbolo`. Al principio iba a poner el nodo nuevo directo en `tabla[pos] = nuevo`, pero me di cuenta de que si ya había alguien en esa cubeta lo iba a borrar (bueno, a perder la referencia). Lo arreglé haciendo `nuevo->sig = tabla[pos]` antes, para que el nuevo apunte al que ya estaba y luego sí lo pongo al inicio.
>
> Otra cosa fue el campo `nombre[16]`. Si alguien me manda un nombre más largo, con `strcpy` me salgo del arreglo. Por eso primero copio el nombre con `strncpy` a una variable local y le pongo el `'\0'` a mano al final.
>
> Para revisar duplicados solo recorro la cubeta donde cae el nombre, porque si el nombre es el mismo, el hash también es el mismo y siempre cae en la misma cubeta. No hace falta buscar en toda la tabla.

---

## 2. Fase 2: El Científico (Ingeniería de Requerimientos C-R-E-O)

### 2.1 Matriz de Especificación Formal
| Elemento C-R-E-O | Definición Técnica |
| :--- | :--- |
| **Contexto (C)** | C99 / GCC o Rust 2021 sin bibliotecas no estándar |
| **Rol (R)** | Ingeniero de software de sistemas y compiladores |
| **Especificación (E)** | Hash djb2, inserción sin duplicados y encadenamiento de colisiones |
| **Output (O)** | Solo código modular y análisis de complejidad |

### 2.2 Prompt Estructurado
```text
CONTEXTO: Estoy haciendo un laboratorio de compiladores en C (C11, GCC) y tengo que
hacer la tabla de símbolos con una tabla hash con encadenamiento. Se compila con
gcc -std=c11 -Wall -Wextra -Werror, así que no puede tener warnings. Solo librerías
estándar. Mi archivo se compila junto con un harness que ya trae main(), por eso
NO pongas main(). Ojo: en el struct el campo nombre es char nombre[16].

ROL: Actúa como ingeniero de software de sistemas y compiladores.

ESPECIFICACIÓN: Respeta exactamente esto de api_sesion11.h:

typedef struct Simbolo {
    char nombre[16];
    int tipo;
    int offset;
    struct Simbolo *sig;
} Simbolo;

unsigned int hash_djb2(const char *clave);
int insertar_simbolo(Simbolo *tabla[], unsigned int cubetas, const char *nombre,
                     int tipo, int offset);

Reglas:
- hash_djb2: empieza en 5381 y hace hash = hash * 33 + c por cada carácter.
  hash_djb2("") = 5381, hash_djb2("abc") = 193485963.
- insertar_simbolo: cubeta = hash_djb2(nombre) % cubetas.
- Si el nombre ya existe regresa 0 y no crea nodo.
- Si es nuevo crea el nodo con malloc, lo encadena y regresa 1.
- Las colisiones se encadenan, nunca se sobrescribe un nodo.
- Si tabla es NULL, nombre es NULL o cubetas es 0, regresa -1.
- Copia el nombre sin pasarte de los 16 bytes (no uses strcpy directo).

OUTPUT: Solo el código de ia_sesion11_c.c y al final una línea con la complejidad.
```

---

## 3. Código Devuelto por el Modelo
```c
#include "api_sesion11.h"
#include <stdlib.h>
#include <string.h>

unsigned int hash_djb2(const char *clave) {
    unsigned int hash = 5381;
    if (clave == NULL) return hash;
    for (const unsigned char *p = (const unsigned char *)clave; *p; p++) {
        hash = ((hash << 5) + hash) + *p; /* hash * 33 + c */
    }
    return hash;
}

int insertar_simbolo(Simbolo *tabla[], unsigned int cubetas, const char *nombre, int tipo, int offset) {
    if (tabla == NULL || nombre == NULL || cubetas == 0) {
        return -1;
    }

    /* copia segura: el campo nombre solo tiene 16 bytes */
    char clave[sizeof(((Simbolo *)0)->nombre)];
    strncpy(clave, nombre, sizeof(clave) - 1);
    clave[sizeof(clave) - 1] = '\0';

    unsigned int idx = hash_djb2(clave) % cubetas;

    for (Simbolo *s = tabla[idx]; s != NULL; s = s->sig) {
        if (strcmp(s->nombre, clave) == 0) {
            return 0; /* duplicado */
        }
    }

    Simbolo *nuevo = (Simbolo *)malloc(sizeof(Simbolo));
    if (nuevo == NULL) return -1;

    memcpy(nuevo->nombre, clave, sizeof(clave));
    nuevo->tipo = tipo;
    nuevo->offset = offset;
    nuevo->sig = tabla[idx]; /* insercion al frente de la cadena */
    tabla[idx] = nuevo;
    return 1;
}

/* Complejidad: hash O(L); insertar O(L + k), k = largo de la cadena de la cubeta. */
```

---

## 4. Fase 3: El Auditor (Resultados del Test Harness)
*Pruebas ejecutadas:* Hash djb2, duplicados, colisiones en una sola cubeta y argumentos inválidos

### Salida del harness
```text
=== AUDITORIA SESION 11 - Tabla hash con encadenamiento ===
Caso                               esperado:              obtenido:              resultado
[ 1] hash_djb2("")                      esperado: 5381         obtenido: 5381         OK
[ 2] hash_djb2("abc")                   esperado: 193485963    obtenido: 193485963    OK
[ 3] insertar x dos veces               esperado: 0            obtenido: 0            OK
[ 4] dos claves, una cubeta             esperado: 2            obtenido: 2            OK
[ 5] offset de beta en cadena           esperado: 4            obtenido: 4            OK
[ 6] insertar en tabla NULL             esperado: -1           obtenido: -1           OK

RESULTADO: 6/6 pruebas superadas
TODO EN VERDE. Documenta tu auditoria en auditorias/auditoria_sesion11.md
```

* [x] Caso base y límites superados sin *Crash*.
* [x] No existen accesos fuera de límites ni *Memory Leaks*.
* [x] La complejidad teórica coincide con la implementada.

> Nota: con el esqueleto original (antes de programar) el harness daba FALLA en casi todos los casos, porque `hash_djb2` regresaba 0 e `insertar_simbolo` regresaba -1 siempre.
>
> Sobre los memory leaks: el harness no libera los nodos al terminar, pero eso es del harness y no de mis funciones. Cada nodo que creo queda guardado en la tabla y no se pierde ninguno.

### Pruebas extra que hice yo
| Prueba | Esperado | Obtenido |
| :--- | :--- | :--- |
| `insertar_simbolo(t, 0, "x", 1, 0)` (cero cubetas) | -1 | -1 |
| `insertar_simbolo(t, 8, NULL, 1, 0)` | -1 | -1 |
| Nombre de 30 letras `"variable_super_larga_de_prueba"` | 1 y se guarda cortado | 1, guarda `"variable_super_"` |
| El mismo nombre largo otra vez | 0 | 0 |

### Reporte Crítico
*¿Qué caso borde o problema omitió la IA y cómo lo solucionaste?*
> La primera vez que le pedí el código a la IA (sin decirle nada del tamaño del nombre) usó `strcpy(nuevo->nombre, nombre);`. Con eso los 6 casos del harness pasaban, porque todos los nombres son cortos ("x", "alpha", "beta"). Pero `nombre` en el struct es de 16 bytes, y si le mandas un nombre más largo, `strcpy` escribe fuera del arreglo y se come la memoria de `tipo`, `offset` y hasta del puntero `sig`.
>
> Lo detecté porque en mi versión del Artesano ya me había fijado en eso. Hice una prueba con un nombre de 30 letras y compilé con `-fsanitize=address`, y me salió un error de `buffer-overflow`.
>
> Lo arreglé agregando al contrato la regla de "no te pases de los 16 bytes, no uses strcpy directo". La segunda versión ya copia con `strncpy` a una variable local y le pone el `'\0'` al final. Otra ventaja es que usa ese mismo nombre cortado para el hash y para comparar, así que si mandas el mismo nombre largo dos veces sí lo detecta como repetido.
>
> Lo que aprendí es que el harness solo prueba nombres cortos, así que estar en verde no significa que no haya un desbordamiento. Hay que pensar en qué pasa con datos que el harness no manda.

---
**Firma digital:** ReyesDiego002
