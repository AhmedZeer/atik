

![image](docs/static/banner.png)
## TL;DR
**Atik** es un acelerador de hardware para transformadores de acoplamiento estrecho (Tightly-Coupled) de código abierto. Construido sobre y para la microarquitectura RocketChip.

*¿Qué lo hace tan especial?* Aquí 👇

✅ Mecanismo de **Atención** fundido directamente en el silicio  
✅ Prototipado de extremo a extremo en **FPGA** (AWS F2)  
✅ Evaluado con cargas de trabajo basadas en **PyTorch**  
✅ Construido sobre la arquitectura **RocketChip** (RISC-V)  
✅ Soporte nativo para **BF16**  
✅ Hasta **225×** más rápido en el mecanismo de atención nativo  
✅ Hasta **96×** más rápido en TinyBERT  
✅ Hasta **50×** más rápido en ViT  
✅ Hasta **30×** más rápido en la fase de prefill de GPT-2  

- Mira estos [videos](https://www.youtube.com/playlist?list=PL6v0daaIvQGsj1eRnxf7YfX6KYgopo65X) para comprender la arquitectura a fondo.
- ¿No crees en los resultados de los benchmarks? Haz clic para ver la [lista de reproducción](https://www.youtube.com/playlist?list=PL6v0daaIvQGvxYVnezbRdfBysHe-s8BjE).
- ¿Quieres simularlo localmente? [Salta aquí](#try-it-now)

*A partir de aquí, la gente nerdy puede continuar leyendo :)*

![image](docs/figs/atik-archi.png)
---
![image](docs/static/compare-chips.png)

## ¿Por qué Atik?
Definitivamente hay un creciente interés en la academia alrededor de los aceleradores de Atención, rivalizando con el interés en el hardware sistólico. Pero honestamente, gran parte de la investigación actual se siente un poco demasiado teórica. Parte de ella depende puramente de simuladores en C++ como gem5, careciendo de simulación con precisión temporal o un flujo VLSI adecuado. De lo contrario, tiende a ser de código cerrado, o una implementación ASIC independiente que realmente no se integra con las cargas de trabajo estándar de CPU. 

Los aceleradores DNN maduros y de código abierto que ya tenemos, como NVDLA y Gemmini, comienzan a sentirse un poco heredados. Están hermosamente construidos para las necesidades de su época, optimizando para CNN y tareas de visión, pero no soportan mecanismos de Atención. Además, se centran en tipos de datos cuantizados como INT8, lo que simplemente no es ideal para las aplicaciones modernas diarias que dependen tanto de BF16. 

Las unidades vectoriales estándar ya no son suficientes. Con los transformadores adoptándose en todas partes, realmente necesitamos matrices sistólicas trabajando junto a módulos Softmax para finalmente abordar esas pesadas cargas de trabajo de atención. 

Para resumirlo todo: lo que la comunidad de hardware de código abierto realmente necesita ahora es un acelerador dedicado de Atención y MatMul. Necesita soportar BF16 nativamente, estar basado en una arquitectura de computadora robusta como RocketChip, ser fácilmente evaluable contra cargas de trabajo modernas de PyTorch y estar totalmente listo para el prototipado en FPGA.

Este es el vacío que **Atik** intenta llenar. Un acelerador de IA de código abierto moderno y de acoplamiento estrecho. Prototipable en FPGA. Verificado en VLSI. ¡Preparado para que alguien lo mande a fabricar (tape-out)!  

![image](docs/static/layout-close-v2.png)

## ¡Pruébalo ahora!

Clona el repositorio Chipyard F2 preparado para Atik en lugar de comenzar desde el Chipyard upstream. Este repositorio ya incluye `generators/atik` y la configuración `build.sbt` de nivel superior que compila las configuraciones de Atik orientadas a Chipyard en `chipyard.jar`.

```bash
git clone https://github.com/AhmedZeer/chipyard-f2.git
cd chipyard-f2
./build-setup.sh riscv-tools
source ./env.sh
```

Utiliza las herramientas conda locales al repositorio antes de ejecutar SBT o los objetivos make del simulador. Esto mantiene SBT usando el mismo Java utilizado por el flujo probado y coloca `dtc` en el `PATH`.

```bash
export JAVA_HOME="$PWD/.conda-env/lib/jvm"
export PATH="$PWD/.conda-env/bin:$JAVA_HOME/bin:$PATH"
java -version
command -v dtc
```

La versión de Java esperada es OpenJDK 20 local al repositorio, ubicada en `.conda-env/lib/jvm`. Si se selecciona Java 21 del sistema en su lugar, SBT puede fallar al cargar la compilación con un error de análisis de Scala como `bad constant pool index`.

Compila un simulador Verilator desde `sims/verilator`. La configuración completa más pequeña es el diseño RoCC 2x2:

```bash
cd sims/verilator
make CONFIG=Atik2x2RoCCConfig
```

Si estás depurando el descubrimiento de configuraciones, el jar del classpath generado debería contener las clases de configuración de Atik después de la primera compilación:

```bash
jar tf ../../.classpath_cache/chipyard.jar | grep "chipyard/Atik2x2RoCCConfig.class"
```

Salida esperada:

```text
chipyard/Atik2x2RoCCConfig.class
```

Compila los binarios de software bare-metal de Atik desde el directorio `software/` de este repositorio:

```bash
cd ../../generators/atik/software
make
```

Finalmente, ejecuta la prueba rápida (smoke test) proporcionada en el simulador Verilator generado:

```bash
cd ../../../sims/verilator
make CONFIG=Atik2x2RoCCConfig run-binary \
  BINARY=../../generators/atik/software/build/atik_smoke.riscv \
  LOADMEM=1
```


## Cómo se calcula la aceleración (speedup)

Para los benchmarks independientes de matmul y atención, la aceleración se mide ejecutando el mismo kernel dos veces: una con la implementación de referencia BF16 en software y otra con Atik. Ambas rutas se cronometran con el contador de ciclos RISC-V, y el resultado de hardware también se verifica contra la salida del software.

```c
const uint64_t cpu_start = atik_bench_read_cycles();
atik_ref_matmul_bf16(...);
const uint64_t cpu_cycles = atik_bench_read_cycles() - cpu_start;

atik_clear_counters();
atik_matmul_bf16(...);
const uint64_t hw_cycles = atik_read_counter(ATIK_COUNTER_TOTAL_CYCLES);
```

Para las cargas de trabajo derivadas de PyTorch, ejecutar el modelo completo dos veces sería innecesariamente costoso. En su lugar, el benchmark perfila la carga de trabajo completa de la CPU, perfila por separado las partes reemplazadas por Atik y estima la carga de trabajo acelerada como:

```text
accelerated_cycles = cpu_total_cycles - cpu_replaced_cycles + hw_replacement_cycles
```

Esto mantiene la comparación basada en ciclos mientras evita reproducciones repetidas del modelo completo. La implementación se encuentra en [`software/src/pytorch_workload.c`](software/src/pytorch_workload.c), especialmente en la ruta de contabilidad `hybrid_cycles`.

## Arquitectura

![Atik Full Architecture](docs/figs/atik-full.png)

Atik es un acelerador adjunto a RoCC. El software describe una operación con un `atik_desc_t`, envía la dirección del descriptor con `set_desc` e inicia la ejecución con `run`. El hardware obtiene ese descriptor, decodifica si la operación es matmul, atención o atención causal, y la despacha al controlador correspondiente.

Tanto matmul como atención utilizan DMA explícito, búferes de bloques (tile) respaldados por SRAM local, conversión de BF16 a punto fijo, una malla MAC de punto fijo compartida y escritura de vuelta en BF16. Matmul usa la malla para `C += A * B`; la atención reutiliza la misma malla para el cálculo de puntuaciones QK y la acumulación de probabilidad-por-V, con softmax programado por escalar, recíproco y normalización a su alrededor.

Las notas de diseño más detalladas se encuentran en [`manifest/architecture.md`](manifest/architecture.md). Los flujos de operación de extremo a extremo están documentados en [`manifest/scenarios/matmul.md`](manifest/scenarios/matmul.md) y [`manifest/scenarios/attention.md`](manifest/scenarios/attention.md).

![Atik Architecture Playlist](docs/figs/architecture-playlist.png) 
Si estás interesado en un análisis profundo de la arquitectura, puedes ver estos [videos](https://www.youtube.com/playlist?list=PL6v0daaIvQGsj1eRnxf7YfX6KYgopo65X).

## Simulación de tiempo y ciclos precisa en FPGA

Atik está integrado con FireSim para simulación de FPGA con precisión de ciclo en AWS F2. Los archivos de configuración de FireSim bajo [`firesim/`](firesim/) describen las recetas de compilación, las entradas de la base de datos de hardware y la configuración de despliegue para las configuraciones RoCC 2x2, 4x4 y 8x8.

Las entradas AGFI precompiladas se listan en [`firesim/config_hwdb.yaml`](firesim/config_hwdb.yaml). Las imágenes nuevas pueden reconstruirse desde las recetas en [`firesim/config_build_recipes.yaml`](firesim/config_build_recipes.yaml) y seleccionarse a través de [`firesim/config_build.yaml`](firesim/config_build.yaml). En la práctica, esto significa que el mismo diseño Chisel puede llevarse desde el código fuente a una imagen de FireSim y evaluarse con las cargas de trabajo de software de este repositorio.


## Resultados de los Benchmarks
![image](docs/figs/aggregate-speedup.png)
![image](docs/figs/peek-speedup.png)

Estos gráficos resumen el mismo conjunto de benchmarks a través de las configuraciones Atik 2x2, 4x4 y 8x8. Las cargas de trabajo son intencionalmente diferentes: matmul estresa la multiplicación de matrices densa BF16, la atención estresa la ruta QK, softmax y PV, mientras que ViT, TinyBERT y GPT-2 prefill ponen a prueba capas de transformadores derivadas de PyTorch con diferentes longitudes de secuencia, anchos de incrustación y contajes de cabezas. El tamaño del problema tiene un efecto directo en la relación de aceleración, ya que los casos más grandes suelen amortizar más eficazmente la gestión de descriptores, la configuración de DMA y la sobrecarga del controlador. Por lo tanto, el conjunto de benchmarks incluye una gama de casos pequeños y grandes, especialmente en las cargas de trabajo derivadas de PyTorch, para que los resultados muestren tanto el comportamiento dominado por sobrecarga como el dominado por computación.

El gráfico agregado suma todos los ciclos de RocketCore y todos los ciclos de Atik para los tamaños de problema coincidentes en cada carga de trabajo, y luego informa `sum(cpu_cycles) / sum(hw_cycles)`. Esta es la visión más conservadora porque cada caso incluido contribuye al número final. En esta vista, el diseño 8x8 alcanza aproximadamente 31x en matmul, 182x en atención, 44x en ViT, 36x en TinyBERT y 32x en GPT-2 prefill.

El gráfico de pico muestra el mejor caso individual observado para cada carga de trabajo y tamaño de hardware. Esto destaca dónde el acelerador amortiza más eficazmente las sobrecargas de control, DMA y configuración sobre bloques de computación más grandes. La aceleración máxima alcanza 225.8x en atención con el diseño 8x8, 96.0x en TinyBERT, 50.5x en ViT, 36.3x en GPT-2 prefill y 31.5x en matmul independiente.

### Cargas de trabajo de PyTorch
Las cargas de trabajo estilo PyTorch se portan a pequeños modelos de referencia en C que preservan la misma estructura de kernel utilizada por la etapa correspondiente del modelo. El benchmark compara tres rutas: la salida original de referencia (ground-truth), la salida de referencia en C y la salida de Atik. Informar tanto el error de CPU como el de hardware asegura que la carga de trabajo en C coincida con el comportamiento previsto del modelo antes de juzgar el resultado del acelerador.

La atención multi-cabeza se maneja descomponiendo el modelo en operaciones QK, softmax y PV por cabeza. Atik acelera esos kernels de atención repetidos mientras el entorno de software mantiene la contabilidad del modelo circundante, la disposición de tensores y la contabilidad a nivel de carga de trabajo consistentes entre configuraciones.

#### TinyBERT
![TinyBERT speedup](docs/figs/tinybert-speedup.png)
TinyBERT mantiene la estructura del transformador pero utiliza una configuración más pequeña que el BERT completo: menos capas, dimensiones ocultas más pequeñas, menos cabezas y un cabezal de clasificador compacto. En estos benchmarks, la longitud de secuencia, el ancho del modelo, el número de cabezas, el tamaño oculto y el número de clases son los parámetros principales que varían.

En las etiquetas de la figura, `S` es la longitud de la secuencia, `D` es la dimensión del modelo, `H` es el número de cabezas de atención y `C` es el número de clases de salida. Aumentar `S` agranda las matrices de atención, aumentar `D` incrementa el trabajo de proyección y matmul, aumentar `H` añade más instancias de atención por cabeza y aumentar `C` expande la proyección del clasificador final.

Los resultados de TinyBERT muestran la sensibilidad al tamaño esperada. Los casos pequeños se ven más afectados por la sobrecarga fija de lanzamiento, DMA y control, mientras que los casos más grandes ofrecen a los diseños 4x4 y 8x8 más trabajo útil por descriptor. La configuración 8x8 escala mejor en los casos grandes de TinyBERT y alcanza el pico más alto observado para TinyBERT.

#### Vision Transformer
![ViT speedup](docs/figs/vit-speedup.png)
Las cargas de trabajo de ViT varían la longitud de secuencia derivada de imagen, la dimensión del parche, el ancho del modelo, el número de cabezas, el tamaño oculto del MLP y el número de clases. Esto hace que ViT sea un benchmark útil para verificar cómo se comporta el acelerador cuando los tamaños de atención y proyección crecen juntos.

Para ViT, `S` es la longitud de la secuencia de tokens después del parcheo, `D` es la dimensión del modelo, `H` es el número de cabezas de atención y `C` es el número de clases de salida. Un `S` más grande aumenta los tamaños de QK y las puntuaciones de atención, un `D` más grande incrementa el trabajo de proyección y acumulación de valores, un `H` más grande aumenta el número de cabezas de atención a programar y un `C` más grande incrementa la capa de clasificación final.

Al igual que con TinyBERT, los casos más grandes de ViT brindan a las configuraciones más anchas más oportunidades para amortizar la sobrecarga. El diseño 8x8 se beneficia constantemente de los tamaños de problema más grandes, mientras que los diseños más pequeños aún mejoran respecto a RocketCore pero se saturan antes.

#### Fase de Prefill de GPT-2
![GPT-2 prefill speedup](docs/figs/gpt2-speedup.png)
El prefill de GPT-2 estresa la ruta del transformador antes de la decodificación autoregresiva token por token. El benchmark varía la longitud de secuencia, el ancho del modelo, el número de cabezas, el tamaño oculto y el tamaño de proyección de vocabulario, por lo que captura el costo de llenar la ventana de contexto inicial.

En las etiquetas de prefill de GPT-2, `S` es la longitud de la secuencia, `D` es la dimensión del modelo, `H` es el número de cabezas de atención y `V` es el tamaño del vocabulario para la proyección de salida. Aumentar `S` agranda la ventana de contexto y el trabajo de atención, aumentar `D` incrementa las proyecciones del transformador, aumentar `H` añade más trabajo de atención por cabeza y aumentar `V` expande la proyección de logits.

Aparece el mismo patrón de escalado aquí: los tamaños de secuencia y modelo más grandes mejoran la utilización del acelerador, y la configuración 8x8 produce las aceleraciones más fuertes. La diferencia entre configuraciones es más clara en los casos de prefill más grandes, donde la malla más ancha tiene suficiente trabajo para mantenerse ocupada.


### Cargas de trabajo en C
![Attention speedup](docs/figs/attention-speedup.png)
![Matmul speedup](docs/figs/matmul-speedup.png)
Las cargas de trabajo independientes en C aíslan los kernels de manera más directa que las pruebas derivadas de PyTorch. Tanto matmul como atención muestran un escalado claro de 2x2 a 4x4 a 8x8, especialmente a medida que el tamaño del problema se vuelve lo suficientemente grande como para ocultar las sobrecargas fijas.

Para la atención, `Q` es el número de tokens de consulta, `KV` es el número de tokens de clave/valor, `D` es la dimensión de la cabeza de clave/consulta y `V` es la dimensión de valor. El cálculo de puntuaciones QK escala con `Q * KV * D`, mientras que la acumulación PV escala con `Q * KV * V`, por lo que aumentar cualquiera de estas dimensiones incrementa la cantidad de trabajo del acelerador.

El escalado no es ilimitado. Una vez que la malla está bien utilizada, el rendimiento comienza a estabilizarse porque el tráfico de DMA, el almacenamiento en búfer local, la programación del controlador y el trabajo de softmax escalar se vuelven más visibles. La atención causal también es más difícil de acelerar que la atención no causal porque el enmascaramiento elimina trabajo útil de parte de la matriz de puntuaciones mientras que el flujo de control aún debe preservar la estructura de dependencia causal.

Para matmul, `M`, `N` y `K` describen la multiplicación `C[M, N] = A[M, K] * B[K, N]`. Aumentar `M` o `N` incrementa la cantidad de bloques de salida, mientras que aumentar `K` incrementa el trabajo de reducción por elemento de salida. El trabajo aritmético total escala como `M * N * K`.

## Videos de Benchmarks
![image](docs/figs/benchmark-playlist.png)
Revisa esta [lista de reproducción](https://www.youtube.com/playlist?list=PL6v0daaIvQGvxYVnezbRdfBysHe-s8BjE) para ver todos los videos de benchmarking que se han realizado en FPGA AWS F2.

## Flujo VLSI
![image](docs/static/compare-masked.png)

Atik también incluye un flujo Hammer/OpenROAD para experimentos de síntesis independientes de `AtikCore`. Los scripts actuales apuntan al núcleo del acelerador en lugar de un RocketTile o ChipTop completo, lo que mantiene el flujo enfocado en la ruta de datos del acelerador, controladores, lógica de DMA y almacenamiento en búfer local. Los scripts de lanzamiento y la configuración de Sky130/OpenROAD se encuentran en [`vlsi/hammer/`](vlsi/hammer/), y el resumen de síntesis más reciente se guarda en [`vlsi/syn.rpt`](vlsi/syn.rpt).

| Métrica | 2x2 normal | 4x4 normal | 8x8 normal |
|---|---:|---:|---:|
| Configuración | `Atik2x2RoCCConfig` | `Atik4x4RoCCConfig` | `Atik8x8RoCCConfig` |
| Tamaño de malla | 2x2 | 4x4 | 8x8 |
| Carriles MAC | 4 | 16 | 64 |
| Células sintetizadas totales | 114,575 | 190,722 | 477,464 |
| Área sintetizada total | 721,211.7 | 1,217,183.6 | 3,130,859.0 |
| Área por carril MAC | 180,302.9 | 76,074.0 | 48,919.7 |

El área escala con el tamaño de la malla, pero el área por carril MAC mejora a medida que las sobrecargas fijas de control y DMA se amortizan en más carriles. Este es el comportamiento esperado para un diseño donde la ruta de descriptores, contadores, unidades de softmax escalar y lógica de DMA de bloques se comparten alrededor de una malla más grande.

| Módulo | Área 2x2 | Área 4x4 | Área 8x8 |
|---|---:|---:|---:|
| `AttentionController` | 192,507.1 | 232,010.0 | 369,431.8 |
| `MatmulController` | 127,462.2 | 133,305.4 | 154,623.3 |
| `AttentionScalarMul` | 99,010.0 | 99,010.0 | 99,010.0 |
| `TileDmaReader` | 81,989.9 | 81,942.3 | 81,908.6 |
| `TileDmaWriter` | 5,700.5 | 9,265.1 | 17,186.5 |

Los bloques más grandes son los controladores de atención y matmul, seguidos por la ruta de multiplicación escalar compartida y el lector de DMA de bloques. Algunos bloques, como `AttentionScalarMul` y `TileDmaReader`, permanecen casi constantes a través de los tamaños de malla porque son infraestructura compartida en lugar de computación por carril.

## Línea de tiempo del proyecto

Atik es la versión actual del proyecto. Es un acelerador de IA modular, sintetizable y adjunto a RoCC con soporte para matmul BF16 y atención en línea, DMA explícito, búferes de bloques respaldados por SRAM, computación en malla compartida, integración con FireSim y un flujo VLSI independiente.

La rama principal anterior es `girdap`. Esa versión exploró una estructura de acelerador más grande con rutas separadas para atención y matmul. Ayudó a validar la dirección, pero el diseño era más difícil de sintetizar, menos modular y tenía más complejidad de integración que la arquitectura actual de Atik de malla compartida.

Antes de eso, la rama `toyrocc` contenía experimentos tempranos de RoCC y módulos prototipo de softmax/atención. Esas piezas fueron útiles para aprender los problemas de control y numéricos, pero no estaban organizadas como el diseño actual de acelerador reutilizable a nivel de módulo.

## Agradecimientos

Atik se basa en el ecosistema de hardware RISC-V de código abierto. Agradecimiento especial a UC Berkeley y a la comunidad más amplia de Chipyard por Rocket Chip, los patrones de integración RoCC y la infraestructura FireSim. El proyecto también depende de la cadena de herramientas Chisel/CHIPS Alliance, los flujos VLSI Hammer/OpenROAD y el ecosistema PDK abierto Sky130 para la ruta de generación e implementación de hardware.

Las versiones históricas de este trabajo se encuentran en las ramas `toyrocc` y `girdap`. Esos experimentos dieron forma a la arquitectura actual de Atik, especialmente el movimiento hacia un acelerador de malla compartida más limpio con DMA explícito, bloques respaldados por SRAM y un ABI de software más pequeño.

## Citación

Si utilizas Atik en trabajos académicos, por favor cita el repositorio:

```bibtex
@misc{atik,
  author = {Ahmed Zeer},
  title = {Atik: RoCC Based Transformer Accelerator},
  year = {2026},
  url = {https://github.com/AhmedZeer/atik}
}
```
