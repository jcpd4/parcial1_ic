# Apuntes — Prácticas 1 y 2 (Ingeniería de los Computadores, 2026-27)

> Basado en los enunciados de la **Práctica 1** (perspectivas profesionales en aceleración de aplicaciones) y la **Práctica 2** (estudio y propuesta de paralelización de una aplicación). Donde el enunciado solo nombra un concepto, aquí se explica con la definición estándar de la asignatura.

## Índice

1. [Práctica 1: vocabulario de la aceleración de aplicaciones](#1-práctica-1-vocabulario-de-la-aceleración-de-aplicaciones)
2. [Práctica 2: tipos de paralelismo y dependencias](#2-práctica-2-tipos-de-paralelismo-y-dependencias)
3. [Datos, memoria y caché](#3-datos-memoria-y-caché)
4. [Tamaño del problema y medición del rendimiento](#4-tamaño-del-problema-y-medición-del-rendimiento)
5. [Opciones de compilación de GCC y autovectorización](#5-opciones-de-compilación-de-gcc-y-autovectorización)
6. [Ganancia, eficiencia y leyes de rendimiento](#6-ganancia-eficiencia-y-leyes-de-rendimiento)
7. [Problemas de los grifos (Tarea 3) resueltos](#7-problemas-de-los-grifos-tarea-3-resueltos)
8. [Elegir una arquitectura paralela](#8-elegir-una-arquitectura-paralela)
9. [Claves imprescindibles](#9-claves-imprescindibles)
10. [Trampas típicas](#10-trampas-típicas)
11. [Ejercicios de autoevaluación](#11-ejercicios-de-autoevaluación)
12. [Soluciones](#12-soluciones)

---

## 1. Práctica 1: vocabulario de la aceleración de aplicaciones

El objetivo de la P1 era buscar ofertas de empleo y crear un léxico de tecnologías. En el examen lo útil es **saber qué significa cada término y dónde encaja**.

### 1.1 Clasificación de arquitecturas (taxonomía de Flynn)

| Clase | Flujos de instrucciones / datos | Ejemplo |
|---|---|---|
| **SISD** | 1 instrucción, 1 dato | CPU monociclo clásica |
| **SIMD** | 1 instrucción, muchos datos | Extensiones SSE/AVX/NEON, unidades vectoriales, (GPU en parte) |
| **MISD** | Muchas instrucciones, 1 dato | Casi teórico (sistemas redundantes) |
| **MIMD** | Muchas instrucciones, muchos datos | Multinúcleo, clústeres |

### 1.2 Términos de la lista de la P1

| Término | Qué es | Memoria / modelo |
|---|---|---|
| **Multicore** | Varios núcleos (pocos, potentes) en un mismo chip | Memoria compartida |
| **Many-core** | Decenas–miles de núcleos más simples (p. ej. GPU, Xeon Phi) | Normalmente compartida en el chip |
| **Multiprocesador** | Varios procesadores que **comparten** memoria (UMA o NUMA) | Compartida → hebras, OpenMP |
| **Multicomputador** | Varios nodos, cada uno con **su propia** memoria, unidos por red (clúster) | Distribuida → paso de mensajes, MPI |
| **Multithreading** | Varias hebras de ejecución. *Software*: hebras del SO. *Hardware*: SMT/Hyper-Threading (un núcleo ejecuta varias hebras compartiendo recursos) | — |
| **SIMD** | Una instrucción opera sobre un vector de datos (p. ej. 8 floats con AVX de 256 bits) | Paralelismo de datos a nivel de instrucción |
| **GPU** | Procesador many-core para paralelismo de datos masivo; modelo SIMT (muchas hebras ejecutan la misma instrucción) | Memoria propia del dispositivo |
| **CUDA** | Modelo y lenguaje de programación de GPU de NVIDIA (propietario) | GPU NVIDIA |
| **OpenCL** | Estándar abierto para programar dispositivos heterogéneos (CPU, GPU, FPGA) | Portátil |
| **SYCL** | Estándar abierto basado en C++ moderno, de una sola fuente, para sistemas heterogéneos (p. ej. oneAPI/DPC++) | Portátil |
| **OpenMP** | Directivas (`#pragma omp`) para paralelizar en **memoria compartida** | Hebras |
| **MPI** | Biblioteca estándar de **paso de mensajes** para **memoria distribuida** | Procesos |
| **Computación heterogénea** | Combinar distintos tipos de procesadores (CPU + GPU + FPGA + NPU) en un sistema | — |
| **HPC** | Computación de altas prestaciones: supercomputadores, clústeres | — |
| **Acelerador** | Hardware especializado que ejecuta una parte del trabajo más rápido que la CPU | GPU, FPGA, NPU, TPU |
| **FPGA** | Hardware reconfigurable: se programa el circuito (VHDL/Verilog, HLS) | Muy eficiente energéticamente para tareas concretas |
| **NPU** | Unidad de procesamiento neuronal: acelerador para inferencia de IA | Móviles, portátiles, edge |

### 1.3 Tendencias de futuro citadas en la P1

Computación cuántica o híbrida, nuevos aceleradores, procesadores con muchísimos núcleos, IA en el borde (*edge AI*), computación neuromórfica (inspirada en neuronas, por eventos), centros de datos sostenibles (eficiencia energética, rendimiento por vatio), nuevos lenguajes paralelos, memoria distribuida y **arquitecturas desagregadas** (CPU, memoria y aceleradores como recursos separados unidos por red, p. ej. CXL).

### 1.4 Idea de fondo de la P1

- Los perfiles con conocimientos de paralelismo y arquitectura piden **C/C++, CUDA, MPI, OpenMP, Linux, optimización, perfilado** y suelen tener **mejor salario** y **más requisitos** (máster/doctorado, experiencia).
- Se concentran en: fabricantes de chips (NVIDIA, AMD, Intel, ARM…), centros de supercomputación (BSC, EuroHPC), IA, automoción, finanzas, investigación.
- Diferencia clave con las ofertas generalistas: las generalistas piden frameworks web, bases de datos, cloud; no piden conocer el hardware.

---

## 2. Práctica 2: tipos de paralelismo y dependencias

### 2.1 Paralelismo de datos vs. funcional

| | Paralelismo de **datos** | Paralelismo **funcional** (de tareas) |
|---|---|---|
| Definición (enunciado) | La **misma operación** se aplica a la vez a **partes distintas de los datos** | **Operaciones o tareas distintas** se ejecutan a la vez, si sus dependencias lo permiten |
| Ejemplo | Multiplicar matrices: cada hebra calcula unas filas de C | Un programa calcula a la vez un histograma y una FFT sobre la misma imagen ya cargada |
| Escala con | El tamaño de los datos | El número de tareas independientes (suele ser pequeño y fijo) |
| Herramientas | SIMD, GPU, `omp parallel for` | `omp sections`, `omp task`, pipelines |

> La P2 exigía una aplicación con **ambos** tipos. Una multiplicación de matrices sola **no basta** porque solo tiene paralelismo de datos.

### 2.2 Grafo de dependencias

- **Nodos** = tareas (relacionadas con funciones o bucles del código).
- **Aristas** A → B = B no puede empezar hasta que A termine (B usa algo que produce A).
- Tareas **sin camino** entre ellas pueden ejecutarse **a la vez** sin cambiar el resultado.

Ejemplo del enunciado (inicialización, dos cálculos independientes, combinación):

```
        ┌──────────┐
        │  Init    │
        └────┬─────┘
       ┌─────┴─────┐
       ▼           ▼
  ┌─────────┐ ┌─────────┐
  │ Cálc. A │ │ Cálc. B │   ← paralelismo funcional (A ‖ B)
  └────┬────┘ └────┬────┘
       └─────┬─────┘
             ▼
        ┌──────────┐
        │ Combinar │
        └──────────┘
```

Conceptos asociados:
- **Trabajo total (W)**: suma de los tiempos de todas las tareas = tiempo secuencial.
- **Camino crítico (D)**: el camino más largo (en tiempo) del grafo. Ni con infinitos procesadores se baja de D.
- **Ganancia máxima por el grafo** = W / D.
- **Grado de paralelismo**: número máximo de tareas que pueden ejecutarse a la vez.

### 2.3 Tipos de dependencias de datos

Sean dos instrucciones/tareas S1 (antes) y S2 (después):

| Tipo | Nombre | Patrón | ¿Se puede eliminar? |
|---|---|---|---|
| **RAW** | Verdadera / de flujo | S1 escribe x, S2 lee x | **No** (es real) |
| **WAR** | Antidependencia | S1 lee x, S2 escribe x | Sí, renombrando (copia de la variable) |
| **WAW** | De salida | S1 y S2 escriben x | Sí, renombrando |

**Condiciones de Bernstein**: dos tareas son paralelizables si ninguna escribe lo que la otra lee o escribe.

**Dependencias en bucles:**
```c
for (i = 0; i < N; i++) a[i] = b[i] + c[i];      // independiente → paralelizable/vectorizable
for (i = 1; i < N; i++) a[i] = a[i-1] + b[i];    // dependencia entre iteraciones (loop-carried) → NO directo
for (i = 0; i < N; i++) s += a[i];               // reducción: dependencia en s, pero paralelizable con sumas parciales
```

---

## 3. Datos, memoria y caché

### 3.1 Qué hay que analizar (apartado 2.2 de la P2)

- Qué estructuras se **leen o modifican con frecuencia** y cuáles solo en una fase.
- **Patrón de acceso**: secuencial, con salto (*stride*), aleatorio.
- Qué datos son **compartidos** entre tareas (posibles conflictos) y cuáles **privados**.

### 3.2 Jerarquía de memoria y localidad

Registros → L1 → L2 → L3 → RAM → disco. Cada nivel es más grande y más lento.

- **Localidad temporal**: lo que usé ahora lo volveré a usar pronto.
- **Localidad espacial**: si uso `a[i]`, pronto usaré `a[i+1]` (misma línea de caché, típicamente 64 bytes).
- En C las matrices se guardan **por filas** (*row-major*). Recorrer por columnas da *stride* = N y falla mucho en caché.

```c
// Bien (por filas, stride 1)          // Mal (por columnas, stride N)
for (i...) for (j...) s += A[i][j];    for (j...) for (i...) s += A[i][j];
```

Mejoras de organización de datos: intercambio de bucles, **tiling/blocking** (trabajar por bloques que quepan en caché), **SoA vs AoS** (estructura de arrays suele vectorizar mejor que array de estructuras), alinear datos.

### 3.3 Problemas en paralelo

- **Condición de carrera**: varias hebras escriben el mismo dato sin sincronización → resultado incorrecto.
- **Falso compartido (*false sharing*)**: hebras escriben variables **distintas** que están en la **misma línea de caché** → la línea rebota entre núcleos (coherencia) y el rendimiento cae, aunque el resultado sea correcto.
- **Coherencia de caché**: mantener coherentes las copias de un dato en las cachés de distintos núcleos (cuesta tráfico).
- **NUMA**: en multiprocesadores grandes, acceder a memoria de otro socket es más lento.
- **Ancho de banda de memoria**: una aplicación limitada por memoria (*memory-bound*) no escala bien al añadir núcleos; una limitada por cálculo (*compute-bound*) sí.

---

## 4. Tamaño del problema y medición del rendimiento

### 4.1 Escoger el caso de prueba

- Debe tardar **del orden de segundos** (idealmente varios segundos a decenas de segundos).
- **No** casos de micro/nanosegundos: el ruido de medición y el coste de **crear, comunicar y sincronizar** hebras/procesos ocultarían la mejora.
- La aplicación debe ejecutarse **sin intervención** (parámetros por argumentos o fichero de configuración). Si es interactiva hay que hacerla reproducible (p. ej. dos motores de ajedrez jugando entre sí).

### 4.2 Escalado con el tamaño

- Se varía un parámetro (N de una matriz, número de elementos, precisión…) y se representa **tiempo vs. tamaño**.
- Se compara con la complejidad teórica: O(N) lineal, O(N²), O(N³) (matrices: duplicar N → tiempo ×8).
- Desviaciones típicas: **saltos** cuando los datos dejan de caber en L1/L2/L3 → la caché importa.
- Esto mide cómo escala el **secuencial**; la ganancia paralela **aún no se puede medir** porque no hay versión paralela.

### 4.3 Metodología de medición correcta

- Medir **tiempo de pared** (*wall-clock*): `clock_gettime(CLOCK_MONOTONIC)`, `std::chrono::steady_clock` o el `real` de `time`.
- `time ./prog` da **real** (pared), **user** (CPU en modo usuario) y **sys** (CPU en el núcleo del SO). En paralelo, user puede ser **mayor** que real.
- **Repetir** varias ejecuciones y dar media/mediana (y desviación).
- Mismo equipo, mismo caso, misma carga, sin otros programas; indicar CPU, SO, versión de GCC, opciones.
- Decidir y declarar si se incluye la E/S (lectura de ficheros, impresión).
- Ojo con frecuencia variable (turbo, ahorro de energía) y con la primera ejecución (cachés frías).

### 4.4 Makefile exigido

```make
CC = gcc
CFLAGS = -O2 -Wall
all: prog
prog: main.c
	$(CC) $(CFLAGS) -o prog main.c
run: prog
	./prog 2000 config.txt
clean:
	rm -f prog *.o
```
`make` compila, `make run` ejecuta el caso de referencia, `make clean` borra lo generado.

---

## 5. Opciones de compilación de GCC y autovectorización

### 5.1 Niveles de optimización

| Opción | Qué hace |
|---|---|
| `-O0` | Sin optimizar (por defecto). Útil para depurar; **no** sirve como referencia realista de rendimiento |
| `-O1` | Optimizaciones básicas que no aumentan mucho el tiempo de compilación |
| `-O2` | Casi todas las optimizaciones que no sacrifican tamaño ni exactitud. Desde GCC 12 incluye vectorización con un modelo de coste muy conservador |
| `-O3` | `-O2` + optimizaciones agresivas: vectorización completa (`-ftree-vectorize`), desenrollado, *inlining* más agresivo… Puede aumentar el tamaño del código |
| `-Os` | Optimiza para tamaño |
| `-Ofast` | `-O3` + `-ffast-math` (**rompe el estándar IEEE 754**) |

### 5.2 Otras opciones relevantes

| Opción | Efecto |
|---|---|
| `-march=native` | Genera instrucciones para **la CPU donde compilas** (AVX2, AVX-512…). El binario puede **no funcionar** en otra CPU |
| `-mtune=native` | Ajusta la planificación de instrucciones para esa CPU **sin** usar instrucciones nuevas (portable) |
| `-mavx2`, `-msse4.2`… | Activa una extensión SIMD concreta |
| `-ftree-vectorize` | Activa la autovectorización de bucles |
| `-fno-tree-vectorize` | La desactiva (útil para comparar) |
| `-funroll-loops` | Desenrolla bucles (menos saltos, más ILP, código más grande) |
| `-ffast-math` | Permite reordenar operaciones en coma flotante, ignorar NaN/Inf… Más rápido, pero el resultado **puede cambiar** |
| `-flto` | Optimización en tiempo de enlace (entre ficheros) |
| `-fopt-info-vec` / `-fopt-info-vec-optimized` | Informa de los bucles **vectorizados** |
| `-fopt-info-vec-missed` | Informa de los bucles **no** vectorizados y **por qué** |
| `-S` | Genera el ensamblador (`.s`) en vez del objeto |
| `gcc --help=optimizers`, `gcc --help=target` | Lista opciones de optimización y de la arquitectura destino |

### 5.3 Autovectorización

El compilador convierte un bucle escalar en uno que usa instrucciones **SIMD**, procesando varios elementos por instrucción.

```c
for (int i = 0; i < N; i++) c[i] = a[i] + b[i];
// con AVX (256 bits) y float (32 bits): 8 sumas por instrucción
```

**Cómo demostrar que hay vectorización:**
1. Informe del compilador: `gcc -O3 -fopt-info-vec-optimized prog.c` → `prog.c:12:5: optimized: loop vectorized using 32 byte vectors`.
2. Ensamblador (`gcc -S` o `objdump -d`) buscando instrucciones **empaquetadas**:
   - `addps`, `mulps`, `vaddps`, `vfmadd...ps` (*packed single*), `...pd` (*packed double*), `vpaddd` (enteros).
   - Registros `ymm` (256 bits, AVX) o `zmm` (512 bits, AVX-512).
3. Relacionar la instrucción con la línea del código fuente.

**Qué impide vectorizar:**
- Dependencias entre iteraciones (`a[i] = a[i-1] + ...`).
- Posible **aliasing** de punteros (el compilador no sabe si `a` y `b` se solapan) → usar `restrict`.
- Llamadas a funciones no *inlineables*, `if` complicados, accesos no contiguos.
- **Reducciones en coma flotante** sin `-ffast-math`: la suma flotante no es asociativa, así que el compilador no puede reordenarla.
- Número de iteraciones desconocido o bucles con `break`.

### 5.4 Comparar configuraciones

- Elegir una **referencia clara** (p. ej. `-O0` o `-O2`) y calcular ganancia = T_ref / T_config.
- Mismo caso, mismas condiciones, poner siempre el **comando de compilación** completo.
- Explicar las diferencias por: uso de caché, vectorización, instrucciones específicas de la CPU (`-march`).

---

## 6. Ganancia, eficiencia y leyes de rendimiento

### 6.1 Definiciones

```latex
S(p) = \frac{T_{ref}}{T(p)} \qquad E(p) = \frac{S(p)}{p}
```

- **Ganancia (speedup) S**: cuántas veces más rápido. T_ref = tiempo del **mejor secuencial** (o del recurso más rápido, como en los grifos).
- **Eficiencia E**: aprovechamiento de cada recurso; entre 0 y 1 (o 0–100 %).
- Ideal: S = p, E = 1 (**ganancia lineal**).
- S > p = **superlineal** (posible, p. ej. porque al repartir los datos caben en caché).

### 6.2 Ley de Amdahl (problema de tamaño fijo)

Si una fracción **f** del programa es paralelizable y (1 − f) es secuencial:

```latex
S(p) = \frac{1}{(1-f) + \frac{f}{p}} \qquad \lim_{p\to\infty} S(p) = \frac{1}{1-f}
```

Ejemplo: f = 0,9 → S máx = 10, aunque tengas 1000 núcleos.

### 6.3 Ley de Gustafson (problema que crece con p)

```latex
S(p) = (1-f) + f \cdot p
```

Si aumentamos el tamaño del problema con el número de procesadores, la parte paralela domina y la ganancia crece casi linealmente.

### 6.4 Lo que impide la ganancia ideal (sobrecostes)

Crear/destruir hebras o procesos, comunicación, sincronización (barreras, cerrojos), **desequilibrio de carga**, partes secuenciales, contención de memoria, falso compartido. **Granularidad**: tareas demasiado pequeñas → el sobrecoste supera la ganancia.

---

## 7. Problemas de los grifos (Tarea 3) resueltos

**Modelo:** un grifo que tarda t horas tiene caudal 1/t depósitos por hora. Los caudales se suman:

```latex
T = \frac{1}{\sum_i \frac{1}{t_i}} \qquad S = \frac{t_{min}}{T} \qquad E = \frac{S}{n}
```

| Caso | Grifos | Caudal total (dep/h) | Tiempo T | Ganancia S (vs. grifo más rápido) | Eficiencia E = S/n |
|---|---|---|---|---|---|
| 1 | 8 h + 40 h | 1/8 + 1/40 = 6/40 | 40/6 = **6,67 h** (6 h 40 min) | 8 / 6,67 = **1,2** | 1,2 / 2 = **0,60 (60 %)** |
| 2 | 8 h + 8 h | 2/8 | **4 h** | 8 / 4 = **2** | 2 / 2 = **1 (100 %)** |
| 3 | 40 h + 40 h | 2/40 | **20 h** | 40 / 20 = **2** | 2 / 2 = **1 (100 %)** |
| 4 | 8 h + 40 h + 40 h | 1/8 + 2/40 = 7/40 | 40/7 = **5,71 h** (5 h 43 min) | 8 / 5,71 = **1,4** | 1,4 / 3 = **0,47 (46,7 %)** |

**Interpretación (lo que se pide explicar):**
- Casos 2 y 3: recursos **homogéneos** → ganancia lineal y eficiencia 100 %. Es el ideal.
- Caso 3: eficiencia 100 % y aun así es **más lento** (20 h) que un solo grifo de 8 h. **Eficiencia alta ≠ tiempo bajo**; la ganancia es relativa a la referencia de cada caso.
- Casos 1 y 4: recursos **heterogéneos** (como CPU + GPU). Añadir recursos lentos ayuda poco: la eficiencia "baja" porque E = S/n supone que todos los grifos son tan rápidos como el mejor.
- En el caso 1 se obtiene exactamente toda la mejora posible (1,2 es el máximo con esos grifos): la eficiencia baja **no** indica mal uso, sino que el grifo lento aporta poco. Una eficiencia relativa a la capacidad real sería S / (Σ caudales / caudal máx) = 1,2 / 1,2 = 100 %.
- Analogía: cada grifo es un procesador; el depósito es el trabajo; el reparto perfecto del caudal equivale a un **equilibrado de carga perfecto sin sobrecostes**.

---

## 8. Elegir una arquitectura paralela

| Arquitectura | Encaja cuando… | Dificultades |
|---|---|---|
| **Multinúcleo (OpenMP, hebras)** | Memoria compartida, paralelismo moderado (datos y tareas), datos que caben en un nodo | Carreras, falso compartido, ancho de banda de memoria, pocos núcleos |
| **GPU (CUDA, OpenCL, SYCL)** | Paralelismo de datos **masivo** y regular, mucho cálculo por dato | Copias CPU↔GPU (PCIe), divergencia de ramas, poco útil para tareas distintas |
| **Clúster (MPI)** | Problema muy grande que no cabe en un nodo, o se necesita mucho más cómputo | Comunicación por red (latencia), reparto de datos, programación más compleja |
| **Heterogéneo (CPU + GPU/FPGA)** | Partes distintas: control y tareas en CPU, núcleos de cálculo en el acelerador | Equilibrar carga entre recursos de distinta velocidad (¡como los grifos 1 y 4!) |
| **SIMD de la CPU** | Bucles regulares sobre arrays contiguos | Dependencias, aliasing, alineación |

Razonar siempre: qué **ganancia** y **eficiencia** cabe esperar (Amdahl con la fracción paralelizable medida) y qué sobrecostes habrá.

---

## 9. Claves imprescindibles

1. **Datos vs. funcional**: misma operación sobre datos distintos vs. tareas distintas a la vez.
2. **Grafo de dependencias**: arista = "debe terminar antes"; sin camino = paralelizable; **camino crítico** limita la ganancia (S máx = W/D).
3. **RAW es la única dependencia real**; WAR y WAW se eliminan renombrando.
4. **Multiprocesador = memoria compartida (OpenMP)**; **multicomputador = memoria distribuida (MPI)**.
5. **S = T_ref / T_par**, **E = S / p**. Saber calcularlos con números y con caudales.
6. **Amdahl**: la parte secuencial limita: S máx = 1/(1 − f).
7. **Gustafson**: si el problema crece con p, la ganancia escala casi linealmente.
8. **Caso de prueba de segundos**, no micro/nanosegundos (sobrecoste y ruido).
9. **Medir bien**: tiempo de pared, varias repeticiones, mismas condiciones, documentar todo.
10. `-O3` vectoriza; `-march=native` usa instrucciones de tu CPU; `-ffast-math` cambia resultados; `-fopt-info-vec` demuestra vectorización.
11. **Evidencia SIMD**: instrucciones empaquetadas (`ps`/`pd`), registros `ymm`/`zmm`, ligadas a una línea del código.
12. **Localidad**: C guarda matrices por filas; recorrer por columnas destroza la caché.
13. **Falso compartido**: variables distintas en la misma línea de caché.
14. En la P2 la ganancia paralela solo se **estima**; lo que se **mide** es secuencial (tamaños y compiladores).

---

## 10. Trampas típicas

| Trampa | Lo correcto |
|---|---|
| "Veo registros `xmm` en el ensamblador → está vectorizado" | En x86-64 **toda** la coma flotante escalar usa `xmm` (`addss`, `addsd`). Solo es vectorial si es **empaquetada**: `addps`, `vaddpd`, registros `ymm`/`zmm` |
| Calcular la ganancia respecto al grifo/procesador **más lento** | Se calcula respecto al **más rápido** (o al mejor secuencial) |
| Eficiencia 100 % = la configuración más rápida | Caso 3 de los grifos: E = 100 % y tarda 20 h, más que un grifo de 8 h |
| Eficiencia baja = mala paralelización | En sistemas heterogéneos E = S/n baja aunque se aproveche todo |
| Sumar tiempos en vez de caudales | Se suman **caudales** (1/t), no tiempos. 8 h y 40 h juntos no son 48 h ni 24 h |
| Usar `-O0` como referencia y vender una ganancia enorme | `-O0` no es realista; indicar claramente cuál es la referencia |
| `-Ofast`/`-ffast-math` "siempre mejor" | Puede cambiar el resultado numérico; hay que comprobar que la salida es correcta |
| `-march=native` es portable | El binario puede fallar en otra máquina (instrucción ilegal). `-mtune` sí es portable |
| Multiplicación de matrices como problema de la P2 | Solo tiene paralelismo de datos; se pedía también **funcional** |
| Medir una ejecución que dura microsegundos | El sobrecoste de hebras y el ruido dominan; usar casos de segundos |
| Confundir `user` con tiempo real | En paralelo `user` suma el tiempo de todos los núcleos; la ganancia se mide con **real** (pared) |
| "Con infinitos núcleos la ganancia es infinita" | Amdahl: S ≤ 1/(1 − f). Y en el grafo, S ≤ W / camino crítico |
| Una reducción `s += a[i]` no se puede paralelizar | Sí: con sumas parciales (reducción). Pero en coma flotante el compilador no la vectoriza sin `-ffast-math` (asociatividad) |
| Ganancia superlineal = error de medición | Puede ser real por efectos de caché |
| Multicore = many-core | Multicore: pocos núcleos complejos. Many-core: muchos núcleos simples |
| OpenMP para clústeres / MPI solo para clústeres | OpenMP: memoria compartida. MPI: memoria distribuida, aunque también funciona en un solo nodo |
| Falso compartido = condición de carrera | El falso compartido da resultados **correctos** pero lentos; la carrera da resultados **incorrectos** |
| Decir que en la P2 se "ha medido" la ganancia paralela | Solo se **estima**; no hay versión paralela todavía |

---

## 11. Ejercicios de autoevaluación

### Bloque A — Conceptos (respuesta corta)

**A1.** Define paralelismo de datos y paralelismo funcional y pon un ejemplo de cada uno en un programa de procesamiento de imágenes.

**A2.** ¿Qué diferencia hay entre un multiprocesador y un multicomputador? ¿Qué herramienta de programación asociarías a cada uno?

**A3.** Clasifica según Flynn: (a) una unidad AVX, (b) un clúster de 64 nodos, (c) un procesador de 8 núcleos.

**A4.** ¿Por qué se pide que el caso de prueba tarde segundos y no microsegundos?

**A5.** ¿Qué es el falso compartido y en qué se diferencia de una condición de carrera?

**A6.** Explica la diferencia entre `-march=native` y `-mtune=native`.

**A7.** ¿Por qué el compilador no vectoriza por defecto `for(i) s += a[i];` con `s` y `a` de tipo `double`? ¿Cómo se puede conseguir?

**A8.** Nombra tres formas de comprobar que el compilador ha autovectorizado un bucle.

**A9.** ¿Qué es una arquitectura desagregada? ¿Y la computación neuromórfica?

**A10.** Diferencia entre CUDA, OpenCL y SYCL.

### Bloque B — Dependencias

**B1.** Indica el tipo de dependencia (RAW, WAR, WAW) entre cada par:
```c
S1: x = a + b;
S2: c = x * 2;
S3: a = c + 1;
S4: x = 5;
```
Pares: (S1,S2), (S1,S3), (S1,S4), (S2,S3).

**B2.** ¿Qué bucles pueden paralelizarse directamente? Justifica.
```c
(a) for (i=0; i<N; i++)   a[i] = 2*b[i];
(b) for (i=1; i<N; i++)   a[i] = a[i-1] * 2;
(c) for (i=0; i<N; i++)   a[i] = a[i+N];        // a tiene 2N elementos
(d) for (i=0; i<N; i++)   max = (a[i] > max) ? a[i] : max;
```

**B3.** Un programa tiene las tareas siguientes (tiempo en segundos):

| Tarea | Tiempo | Depende de |
|---|---|---|
| A (leer datos) | 2 | — |
| B (filtrar) | 6 | A |
| C (estadísticas) | 4 | A |
| D (histograma) | 3 | A |
| E (combinar B y C) | 2 | B, C |
| F (escribir) | 1 | D, E |

(a) Dibuja el grafo. (b) Calcula el tiempo secuencial. (c) Calcula el camino crítico. (d) ¿Cuál es la ganancia máxima solo con paralelismo funcional? (e) ¿Cuántos procesadores bastan para alcanzarla?

### Bloque C — Rendimiento

**C1.** Un programa tarda 120 s en secuencial y 20 s con 8 núcleos. Calcula la ganancia y la eficiencia.

**C2.** El 80 % de un programa es paralelizable. Según Amdahl, ¿qué ganancia se obtiene con 4 y con 16 procesadores? ¿Cuál es la ganancia máxima?

**C3.** Con el mismo f = 0,8, ¿qué ganancia predice Gustafson con 16 procesadores? ¿Por qué es distinta?

**C4.** Se quiere una ganancia de 10 con 20 procesadores. ¿Qué fracción paralelizable mínima hace falta (Amdahl)?

**C5.** Un programa con matrices N×N tarda 2 s con N = 500 y 16 s con N = 1000. ¿Qué complejidad sugiere? ¿Cuánto tardaría con N = 2000? Si con N = 2000 tarda 200 s, ¿qué puede estar pasando?

**C6.** Tabla de tiempos de compilación:

| Opciones | Tiempo (s) |
|---|---|
| `-O0` | 24,0 |
| `-O2` | 8,0 |
| `-O3` | 6,0 |
| `-O3 -march=native` | 4,0 |

Calcula las ganancias respecto a `-O2` y respecto a `-O0`. ¿Qué referencia es más honesta? ¿Por qué `-march=native` mejora?

### Bloque D — Grifos (variantes)

**D1.** Un grifo tarda 6 h y otro 12 h. Calcula T, S y E.

**D2.** Cuatro grifos: uno de 5 h y tres de 15 h. Calcula T, S y E.

**D3.** Tienes un grifo de 10 h. ¿Cuántos grifos de 30 h tienes que añadir para que el depósito se llene en 5 h? ¿Cuál es la eficiencia resultante?

**D4.** (Trampa) Dos grifos de 8 h, pero se sabe que por un problema de presión, al abrirlos juntos cada uno da el 75 % de su caudal. Calcula T, S y E. ¿Qué representa ese 75 % en un sistema paralelo?

### Bloque E — Ensamblador

**E1.** ¿Cuáles de estas instrucciones son vectoriales (SIMD empaquetadas)? `addsd %xmm1,%xmm0` · `vaddps %ymm1,%ymm2,%ymm0` · `mulss %xmm2,%xmm3` · `vfmadd231pd %zmm1,%zmm2,%zmm0` · `vpaddd %ymm0,%ymm1,%ymm2`

**E2.** Si un bucle sobre `float` se vectoriza con registros `ymm`, ¿cuántos elementos procesa cada instrucción? ¿Y con `double` y `zmm`?

---

## 12. Soluciones

### Bloque A

**A1.** Datos: aplicar el mismo filtro a cada píxel (o a cada franja de la imagen) en paralelo. Funcional: a la vez calcular el histograma y detectar bordes sobre la imagen ya cargada (tareas diferentes e independientes).

**A2.** Multiprocesador: varios procesadores comparten memoria → OpenMP/hebras. Multicomputador: nodos con memoria propia conectados por red → MPI.

**A3.** (a) SIMD. (b) MIMD (memoria distribuida). (c) MIMD (memoria compartida).

**A4.** Porque en tiempos tan cortos el ruido de medición y el coste de crear, comunicar y sincronizar hebras/procesos son comparables o mayores que el trabajo útil; las mejoras no se apreciarían.

**A5.** Falso compartido: hebras escriben variables distintas que están en la misma línea de caché; el protocolo de coherencia hace rebotar la línea → lento pero correcto. Condición de carrera: acceso concurrente sin sincronización al mismo dato con al menos una escritura → resultado incorrecto.

**A6.** `-march=native` usa las instrucciones de la CPU actual (no portable). `-mtune=native` solo ajusta la planificación a esa CPU sin usar instrucciones nuevas (portable).

**A7.** Vectorizar la suma requiere reordenarla (sumas parciales), y la suma en coma flotante no es asociativa: el resultado podría cambiar. Se permite con `-ffast-math` (o `-fassociative-math`), con `-Ofast`, o con `#pragma omp simd reduction(+:s)`.

**A8.** (1) `-fopt-info-vec-optimized` ("loop vectorized"). (2) Ensamblador con `-S`/`objdump -d` buscando instrucciones empaquetadas y registros `ymm`/`zmm`. (3) Comparar tiempos con `-fno-tree-vectorize` frente a vectorizado.

**A9.** Desagregada: CPU, memoria y aceleradores son recursos separados en red que se asignan bajo demanda. Neuromórfica: hardware inspirado en las neuronas, que procesa por eventos (*spikes*) con muy bajo consumo.

**A10.** CUDA: propietario de NVIDIA, solo sus GPU. OpenCL: estándar abierto en C para CPU, GPU, FPGA. SYCL: estándar abierto en C++ moderno de una sola fuente, también heterogéneo.

### Bloque B

**B1.**
- (S1,S2): **RAW** en x (S1 escribe x, S2 lo lee).
- (S1,S3): **WAR** en a (S1 lee a, S3 escribe a).
- (S1,S4): **WAW** en x (los dos escriben x).
- (S2,S3): **RAW** en c (S2 escribe c, S3 lo lee).

**B2.**
- (a) Sí: iteraciones independientes.
- (b) No directamente: dependencia RAW entre iteraciones (`a[i-1]`). (Esta en concreto tiene forma cerrada: a[i] = a[0]·2^i.)
- (c) Sí: lee `a[N..2N-1]` y escribe `a[0..N-1]`, zonas que no se solapan.
- (d) Sí como **reducción** de máximo (máximos parciales y luego el máximo de ellos), no como bucle independiente directo.

**B3.**
(a)
```
          ┌── B(6) ──┐
A(2) ─────┼── C(4) ──┴── E(2) ──┐
          └── D(3) ─────────────┴── F(1)
```
(b) W = 2 + 6 + 4 + 3 + 2 + 1 = **18 s**.
(c) Caminos: A-B-E-F = 2+6+2+1 = **11**; A-C-E-F = 9; A-D-F = 6. Camino crítico = **11 s**.
(d) S máx = 18 / 11 = **1,64**.
(e) Tras A pueden ejecutarse a la vez B, C y D (grado de paralelismo 3), pero **con 2 procesadores basta**:
- P1: A (0–2), B (2–8), E (8–10), F (10–11).
- P2: C (2–6), D (6–9).
- E empieza en 8 (B y C ya han terminado); F empieza en 10 (D terminó en 9) → fin en **11 s** = camino crítico.

Trampa: el grado de paralelismo máximo (3) no siempre es el número de procesadores necesario.

### Bloque C

**C1.** S = 120/20 = **6**; E = 6/8 = **0,75 (75 %)**.

**C2.** p = 4: S = 1/(0,2 + 0,8/4) = 1/0,4 = **2,5**. p = 16: S = 1/(0,2 + 0,05) = **4**. Máxima: 1/0,2 = **5**.

**C3.** S = 0,2 + 0,8·16 = **13**. Gustafson supone que el problema crece con p (la parte paralela crece), Amdahl supone un problema de tamaño fijo.

**C4.** 10 = 1/((1−f) + f/20) → (1−f) + f/20 = 0,1 → 1 − 0,95f = 0,1 → f = 0,9/0,95 ≈ **0,947 (94,7 %)**.

**C5.** Duplicar N multiplica el tiempo por 8 = 2³ → **O(N³)**. N = 2000 → 16·8 = **128 s**. Si tarda 200 s, el rendimiento empeora más de lo esperado: probablemente las matrices **ya no caben en caché** (más fallos, acceso a RAM).

**C6.**

| Opciones | S vs `-O2` | S vs `-O0` |
|---|---|---|
| `-O0` | 0,33 | 1 |
| `-O2` | 1 | 3 |
| `-O3` | 1,33 | 4 |
| `-O3 -march=native` | 2 | 6 |

La referencia `-O2` es más honesta: `-O0` no es una compilación realista. `-march=native` mejora porque usa instrucciones SIMD más anchas (AVX2/AVX-512) y FMA que la arquitectura genérica x86-64 no permite (solo SSE2).

### Bloque D

**D1.** Caudal = 1/6 + 1/12 = 3/12 = 1/4 → T = **4 h**. S = 6/4 = **1,5**. E = 1,5/2 = **0,75**.

**D2.** Caudal = 1/5 + 3/15 = 2/5 → T = **2,5 h**. S = 5/2,5 = **2**. E = 2/4 = **0,5**.

**D3.** Necesitamos caudal 1/5. 1/10 + k/30 = 1/5 → k/30 = 1/10 → **k = 3** grifos. S = 10/5 = 2; n = 4 grifos → E = **0,5**.

**D4.** Caudal = 2 · 0,75 · 1/8 = 3/16 → T = 16/3 = **5,33 h**. S = 8/5,33 = **1,5**. E = 1,5/2 = **0,75**. El 75 % representa la **contención de recursos compartidos** (ancho de banda de memoria, bus, red): al trabajar juntos, cada procesador rinde menos que solo. Es un sobrecoste que impide la ganancia lineal.

### Bloque E

**E1.** Vectoriales: `vaddps` (ymm, packed single), `vfmadd231pd` (zmm, packed double), `vpaddd` (ymm, enteros empaquetados). Escalares: `addsd` y `mulss` (sufijo *s* = *scalar*, aunque usen `xmm`).

**E2.** `ymm` = 256 bits → 256/32 = **8 floats**. `zmm` = 512 bits → 512/64 = **8 doubles**.
