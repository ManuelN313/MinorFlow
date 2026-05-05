# Informe CARLA

## Diseño y Parametrización de un Procesador RISC-V: Arquitectura y Optimizaciones

### Introducción y Motivación del Diseño

A continuación se detalla la microarquitectura de un núcleo de procesamiento RISC-V operando a una frecuencia base de 100 MHz, diseñado e implementado para simulaciones en C y RISC-V assembler. La elección de una arquitectura de ejecución en orden (*in-order*) y emisión simple (*single-issue*) responde a un objetivo fundamental: mantener una base conceptual lo suficientemente accesible para que cualquier persona con conocimientos básicos de arquitectura de computadoras pueda comprender su funcionamiento. 

Sin embargo, sobre esta base clásica, se han integrado optimizaciones microarquitectónicas propias de procesadores modernos de su misma categoría. Esto permite observar cómo técnicas avanzadas de mitigación de latencia y manejo de memoria pueden acoplarse a un diseño fundamentalmente simple para maximizar su rendimiento.

---

### Topología del Pipeline y Elasticidad

El núcleo implementa un cauce (*pipeline*) segmentado clásico de cuatro etapas lógicas principales: Búsqueda (*Fetch*), Decodificación (*Decode*), Ejecución (*Execute*) y Retiro (*Commit*). Para transformar este modelo rígido en un **pipeline elástico**, se han introducido colas (buffers) de desacoplamiento entre las etapas, permitiendo absorber latencias cortas sin estancar el flujo completo.

**Parámetros de Ancho de Banda y Colas:**
* **Ancho de Búsqueda y Emisión (*Fetch & Issue Width*):** Estrictamente se busca (*fetch*) 1 instrucción por ciclo desde la memoria de instrucciones, y de igual manera se decodifica y emite 1 instrucción por ciclo en el resto de las etapas de procesamiento frontal (*Front-end*).
* **Límite de Retiro (*Commit Limit*):** 2 instrucciones por ciclo. Esta asimetría frente a la emisión permite retirar simultáneamente dos instrucciones que finalicen en el mismo ciclo, ya sea combinando una operación de memoria de larga latencia con una operación aritmética rápida, o bien retirando dos operaciones aritméticas al mismo tiempo.
* **Buffers de Desacoplamiento (FIFOs inter-etapa):**
    * `Fetch` a `Decode`: 2 entradas.
    * `Decode` a `Execute`: 4 entradas.
    * **Cola de Ejecución:** 8 entradas (mantiene a las unidades funcionales alimentadas de forma constante).

---

### Segmentación de las Unidades Funcionales y SIMD

Para modelar con precisión el comportamiento de un núcleo moderno con soporte para extensiones vectoriales, se descartó el uso de unidades aritmético-lógicas (ALUs) monolíticas. En su lugar, el sistema divide la ejecución en submódulos especializados con latencias de acierto (*hit latency*) y latencias de emisión (*issue latency*) específicas:

| Unidad Funcional | Operaciones Soportadas | Latencia (Ciclos) | Latencia de Emisión |
| :--- | :--- | :--- | :--- |
| **ALU Simple y SIMD Rápido** | Sumas, Lógica, Desplazamientos, Predicados | 2 | 1 |
| **SIMD Complejo** | MAC, Flotante, Reducciones, Criptografía (AES/SHA) | 4 | 1 |
| **SIMD Matricial** | Multiplicación de Matrices | 6 | 2 |
| **SIMD Div/Sqrt** | División iterativa y Raíz Cuadrada | 15 | 12 |
| **LSU (Load/Store)** | Accesos escalares y vectoriales contiguos | 2 | 1 |

---

### Subsistema de Memoria y Colas de Carga/Escritura (LSQ)

El manejo de la memoria es el componente más fuertemente optimizado del procesador. El núcleo interactúa con un bus de datos de 64 bits (8 bytes por ciclo) y cuenta con una arquitectura de colas avanzada (LSQ - *Load/Store Queue*) que previene cuellos de botella al procesar vectores:

* **Store Buffer Ampliado:** Equipado con 16 entradas. Permite que el procesador despache operaciones de escritura y continúe su ejecución inmediatamente, delegando la escritura física en la caché a la lógica de fondo.
* **Emisión Temprana de Memoria (*Early Memory Issue*):** Habilitada. Las instrucciones de carga (Loads) pueden acceder a la caché L1 antes de ser la instrucción más antigua del pipeline, mitigando la latencia de espera.
* **Jerarquía de Caché L1 Asimétrica y MSHRs:**
    * **L1 Instrucciones:** 16 KiB, Asociatividad de 4 vías, 4 registros MSHR (Miss Status Holding Registers) con 8 *targets* cada uno.
    * **L1 Datos:** 32 KiB, Asociatividad de 8 vías, 8 registros MSHR con 16 *targets* cada uno.
    * *Nota de diseño:* Ambas cachés operan con líneas de 64 bytes para maximizar la localidad espacial y configuraciones de lectura paralela (acceso no secuencial) para garantizar aciertos en 2 ciclos.

---

### Predicción de Saltos

Para mantener un flujo constante de instrucciones en el *Front-end*, se implementó un sistema de predicción de saltos de bajo costo espacial pero alta eficiencia, estructurado en tres componentes:

1.  **Predictor de Dirección (LocalBP):** Tabla de historial local de 1024 entradas, utilizando contadores saturados estándar de 2 bits.
2.  **Branch Target Buffer (BTB):** Memoria caché de 256 entradas con asociatividad de 4 vías, encargada de almacenar las direcciones de destino de los saltos previamente tomados.
3.  **Return Address Stack (RAS):** Pila de hardware de 16 niveles dedicada exclusivamente a predecir las direcciones de retorno de llamadas a funciones anidadas.


![Descripción de la Arquitectura](gem5_minorcpu_arch.png)