# Bitácora de Requerimientos y Auditoría IA — Sesión 03
**Materia:** Lenguajes y Autómatas II (SCD-1016)  
**Tema:** Google Colab, Mágicos y Flujo de Entrega  
**Docente Titular:** MCC. Iván Márquez Larios (`ijmarquezl`)  
**Estudiante:** Diego Reyes Martínez
**Matrícula:** 24531376  
**Track:** [ C ]  

---

## 1. Fase 1: El Artesano (Dificultad Manual)
*Reto:* Manipulación de archivos y llamadas OS  
*Describe qué problema o restricción de memoria/sintaxis identificaste al codificar a mano:*
> Lo que más me costó fue contar bien las líneas en los tres casos raros. Al principio solo contaba los `\n`, y con eso el archivo `"a\nb"` me daba 1 en vez de 2, porque la última línea no tiene salto final. Lo arreglé guardando el último carácter leído con `fgetc`: si no es `\n`, sumo una línea más. Otro detalle fue el archivo vacío, que me daba 1 al principio. Lo resolví iniciando la variable `ultimo` en `'\n'`, así que si el archivo no tiene nada nunca se suma la línea extra. También tuve que acordarme de cerrar el archivo con `fclose` en todos los caminos, incluso cuando `fputs` falla, para no dejar archivos abiertos. Por último, la sintaxis: `fopen` regresa `NULL` cuando la ruta no existe, y si no lo revisaba el programa se caía con un segmentation fault. Por eso reviso eso primero y regreso -1.

---

## 2. Fase 2: El Científico (Ingeniería de Requerimientos C-R-E-O)

### 2.1 Matriz de Especificación Formal
| Elemento C-R-E-O | Definición Técnica |
| :--- | :--- |
| **Contexto (C)** | C99 / GCC o Rust 2021 sin bibliotecas no estándar |
| **Rol (R)** | Ingeniero de software de sistemas y compiladores |
| **Especificación (E)** | Salida estándar y códigos de retorno ABI |
| **Output (O)** | Solo código modular y análisis de complejidad |

### 2.2 Prompt Estructurado
```text
Contexto: Estoy en la materia Lenguajes y Autómatas II. Trabajo en C y compilo con gcc -std=c11 -Wall -Wextra -Werror, así que cualquier advertencia cuenta como error. Ya existe un archivo api_sesion03.h con estas firmas:
int escribir_texto(const char *ruta, const char *texto);
int contar_lineas(const char *ruta);

Rol: Actúa como ingeniero de software de sistemas y compiladores, con experiencia en manejo de archivos y códigos de retorno.

Especificación:
- escribir_texto escribe el texto en la ruta indicada. Regresa 0 si todo salió bien y -1 si no se pudo abrir o escribir el archivo.
- contar_lineas regresa cuántas líneas tiene el archivo. Un archivo vacío tiene 0 líneas, y una última línea sin salto de línea también cuenta. Si el archivo no existe regresa -1.
- Cierra siempre el archivo con fclose en todos los caminos.
- No uses main(), no cambies las firmas y no uses variables globales.

Output: Dame solo el código del archivo ia_sesion03_c.c, con comentarios breves, y al final un análisis corto de la complejidad en tiempo y en espacio.
```

---

## 3. Código Devuelto por el Modelo
```c
// Código devuelto por el LLM
#include "api_sesion03.h"
#include <stdio.h>
#include <string.h>

int escribir_texto(const char *ruta, const char *texto) {
    FILE *f = fopen(ruta, "w");
    if (f == NULL) {
        return -1;
    }

    size_t largo = strlen(texto);
    /* fwrite regresa cuantos bytes escribio, si no son todos hubo error */
    if (fwrite(texto, 1, largo, f) != largo) {
        fclose(f);
        return -1;
    }

    if (fclose(f) != 0) {
        return -1;
    }
    return 0;
}

int contar_lineas(const char *ruta) {
    FILE *f = fopen(ruta, "r");
    if (f == NULL) {
        return -1;
    }

    char buffer[256];
    int lineas = 0;
    int linea_abierta = 0; /* 1 si estoy a la mitad de una linea sin cerrar */

    /* fgets lee por pedazos, asi que una linea larga puede venir en varios */
    while (fgets(buffer, sizeof(buffer), f) != NULL) {
        size_t n = strlen(buffer);
        if (n > 0 && buffer[n - 1] == '\n') {
            lineas++;
            linea_abierta = 0;
        } else {
            linea_abierta = 1;
        }
    }

    /* si el archivo termino a la mitad de una linea, tambien cuenta */
    if (linea_abierta) {
        lineas++;
    }

    fclose(f);
    return lineas;
}

/* Complejidad: tiempo O(n) con n = numero de caracteres del archivo,
   porque se recorre una sola vez. Espacio O(1), solo se usa un buffer fijo
   de 256 bytes sin importar el tamaño del archivo. */
```

---

## 4. Fase 3: El Auditor (Resultados del Test Harness)
*Pruebas ejecutadas:* Detección de errores de compilación GCC

* [x] Caso base y límites superados sin *Crash*.
* [x] No existen accesos fuera de límites ni *Memory Leaks*.
* [x] La complejidad teórica coincide con la implementada.

### Reporte Crítico
*¿Qué caso borde o problema omitió la IA y cómo lo solucionaste?*
> En su primera respuesta la IA contaba una línea por cada llamada a `fgets`. Eso funciona con líneas cortas, pero si una línea mide más de 255 caracteres, `fgets` la parte en varios pedazos y la IA la contaba como varias líneas. El harness de la sesión no prueba ese caso, así que lo detecté revisando el código y pensando en qué pasa cuando el buffer se llena. Lo corregí revisando si cada pedazo termina en `\n`: solo en ese caso cuento una línea completa, y con la bandera `linea_abierta` cuento la última línea si el archivo termina sin salto final. También revisé que usara `fclose` en todos los caminos de error y que no cambiara las firmas del contrato. Después compilé con `-Wall -Wextra -Werror` y el harness salió 6/6 en verde.

---
**Firma digital:** [ReyesDiego002]
