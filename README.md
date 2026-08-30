# Post-contenido — Unidad 2: Patrones Creacionales
## Descripción
Repositorio del post-contenido de la Unidad 2 de Patrones de Diseño
de Software — Sexto Semestre. Un único proyecto Maven
(exportador-reportes/) que resuelve la exportación de reportes
académicos en múltiples formatos (Parte 1) y se extiende con
configuración compleja y evaluación de Singleton (Parte 2).


## Cómo ejecutar
```bash
$ cd exportador-reportes
$ mvn compile
```
## Decisiones de diseño

### Decisión 1 — Factory Method vs. Abstract Factory (Parte 1)

**Patrón elegido:** Abstract Factory

## Análisis mediante Preguntas Diagnósticas

| Pregunta Diagnóstica | Análisis y Respuesta para el Sistema de Reportes | Patrón al que apunta |
| :--- | :--- | :--- |
| **1. ¿El sistema crea un único tipo de producto que varía por formato, o varios productos relacionados que deben mantenerse coherentes entre sí dentro del mismo formato?** | El sistema crea **dos productos relacionados**: el cuerpo (tabla de notas) y el encabezado/pie de página. Ambos pertenecen a un mismo formato y deben mantenerse coherentes entre sí para no dañar el documento. | **Abstract Factory** (Familia completa de productos) |
| **2. Al agregar el futuro formato CSV, ¿basta con agregar una nueva implementación de un único producto, o hay que agregar una familia completa de piezas relacionadas?** | Se requiere agregar una **familia completa de piezas relacionadas**: una versión en CSV para el cuerpo y otra versión en CSV para el encabezado/pie de página. | **Abstract Factory** (Familia nueva completa) |
| **3. ¿El riesgo real del problema es "se instancia la clase equivocada" o es "se mezclan piezas de familias distintas y el documento queda inconsistente"?** | El riesgo real del problema es **mezclar piezas de familias o formatos distintos** (por ejemplo, combinar un cuerpo en Excel con un encabezado en PDF), lo cual generaría un documento inconsistente e inválido. | **Abstract Factory** (Riesgo de mezclar familias) |

## Justificación y Descarte de la Alternativa

* **Por qué se eligió Abstract Factory:** Garantiza por diseño que todas las piezas que conforman el reporte (cuerpo, encabezado y pie de página) se creen juntas bajo un mismo formato compatible. Al centralizar la creación en una fábrica por formato, se imposibilita estructuralmente que el sistema combine un cuerpo de un tipo con un encabezado de otro.
* **Por qué se descartó Factory Method:** Factory Method está pensado para crear **un solo producto aislado**. Si usáramos Factory Method, tendríamos un creador separado para el cuerpo y otro creador separado para el encabezado, dejando en el código principal la responsabilidad de no cruzarlos. Esto mantendría abierto el riesgo de mezclar formatos distintos, fallando en resolver el problema central del ejercicio.

### Decisión 2 — Mecanismo de extensibilidad de formatos (Parte 1)


**Opción elegida:** Registro Dinámico con `Map<String, Supplier<ReportFormatFactory>>` (Lookup Table)

## Justificación de Diseño y Cumplimiento del Principio OCP

1. **Eliminación de Control de Flujo Rígido (`if/else` o `switch`):**
   - En lugar de evaluar condicionales rígidos sobre la cadena de texto con el nombre del formato, se utiliza un diccionario (`Map`) que asocia cada identificador de formato (`"pdf"`, `"excel"`, `"html"`) con un proveedor funcional (`Supplier<ReportFormatFactory>`).

2. **Extensibilidad sin Modificación (Open/Closed Principle - OCP):**
   - Cuando se agregue el formato **CSV** en el futuro, no será necesario modificar el método de resolución ni tocar el código cliente existente. Bastará con registrar la nueva fábrica concreta (`CsvReportFactory`) en el mapa de registro (mediante un bloque estático o un método `registerFactory`).

3. **Coherencia con Abstract Factory:**
   - Este mecanismo no reemplaza el patrón Abstract Factory; al contrario, actúa como un localizador/resolvedor de fábricas que retorna la instancia de la fábrica concreta adecuada (`ReportFormatFactory`). Una vez obtenida la fábrica desde el `Map`, el cliente la utiliza para crear la familia completa de productos (`ReportBody` y `ReportHeaderFooter`) garantizando la consistencia del documento.

### Decisión 3 — Builder vs. constructor telescópico vs. setters (Parte 2)


**Patrón elegido:** Builder (Patrón Creacional — GoF)

---

## Análisis Diagnóstico y Descarte de Alternativas Tradicionales

| Alternativa Evaluada | Problema Técnico y Riesgo de Diseño | Veredicto |
| :--- | :--- | :--- |
| **1. Constructor único con 9 parámetros** | **Fragilidad y errores silenciosos:** Obliga a recordar el orden exacto de 9 argumentos. Al tener múltiples valores del mismo tipo (`String`, `boolean`), un intercambio accidental no genera error de compilación y provoca fallos en tiempo de ejecución. | **Descartado** |
| **2. Constructores sobrecargados ("Telescópicos")** | **Explosión combinatoria e inmantenibilidad:** Con 8 parámetros opcionales, el número de combinaciones crece rápido. Lleva al antipatrón de _constructor telescópico_, generando código repetitivo y difícil de leer. | **Descartado** |
| **3. Clase mutable con setters sueltos (JavaBeans)** | **Estados inconsistentes y pérdida de inmutabilidad:** El objeto nace "a medio configurar" y expuesto a estados inválidos antes de invocar todos los setters. Además, imposibilita tener un punto único de validación atómica. | **Descartado** |

---

## Justificación de la Elección del Patrón Builder

La adopción del patrón **Builder** resuelve la construcción de `ExportConfig` mediante tres pilares fundamentales:

1. **Legibilidad e Intencionalidad en el Código Cliente:**  
   Sustituye la invocación a ciegas de un constructor por una interfaz de métodos encadenados (*Fluent API*) con nombres expresivos (por ejemplo, `.withPageSize("A4")`, `.withWatermark("CONFIDENCIAL")`), haciendo explícito cada valor configurado.

2. **Garantía de Inmutabilidad (*Thread-Safety*):**  
   El constructor de `ExportConfig` se define privado y solo es accesible mediante la clase anidada `Builder`. Una vez invocado el método `.build()`, se retorna una instancia totalmente **inmutable** (campos `final` y sin setters), garantizando su seguridad en entornos concurrentes.

3. **Validación Atómica y Valores por Defecto Centralizados:**  
   El método `.build()` actúa como el **único punto de control de calidad**. Permite validar combinaciones requeridas (asegurando el parámetro obligatorio `format`) y asignar automáticamente los valores por defecto razonables a los 8 parámetros opcionales omitidos por el usuario.

### Decisión 4 — ¿ReportFactoryRegistry necesita ser Singleton? (Parte 2)

**Conclusión:** **NO** conviene convertir `ReportFactoryRegistry` en un Singleton clásico. La implementación actual como **clase utilitaria final con miembros y métodos estáticos** es la opción adecuada y correcta para este sistema.

---

### Justificación Basada en los Cuatro Criterios Objetivos

#### 1. Criterio: Identidad de Objeto

* En `ReportExportService`, el único cliente del registro, se invoca `ReportFactoryRegistry.resolve(format)` de forma estática. No hay en ningún lugar del código una necesidad de polimorfismo: el registro no implementa una interfaz `Registry`, no se pasa como parámetro de constructor a otros objetos, ni se inyecta vía *Dependency Injection*. El servicio simplemente llama a los métodos estáticos directamente.

En las pruebas unitarias, si fuera necesario mockear el comportamiento del registro, se pueden usar herramientas que manejan métodos estáticos (como Mockito moderno). Sin embargo, en la práctica, mockeando las fábricas concretas (`PdfReportFactory`, `ExcelReportFactory`, `HtmlReportFactory`) se consigue el objetivo sin necesidad de tratar el registro como un objeto polimórfico.
* **Conclusión del criterio:** **No hay necesidad de "identidad de objeto".** Los métodos estáticos son suficientes y directos.

---

#### 2. Criterio: Inicialización Costosa

* El bloque `static { ... }` de `ReportFactoryRegistry` contiene tres asignaciones simples mediante referencias a constructores:

```java
  static {
      REGISTRY.put("pdf", PdfReportFactory::new);
      REGISTRY.put("excel", ExcelReportFactory::new);
      REGISTRY.put("html", HtmlReportFactory::new);
  }
```
 Esto se ejecuta una sola vez cuando Java carga la clase. Al no leer archivos ni conectarse a bases de datos, toma menos de un milisegundo. No se gana nada posponiendo esta carga con *lazy loading* o `getInstance()`. Al contrario, usar `static` garantiza que la carga sea rápida y predecible desde el primer momento.
* **Conclusión del criterio:** **La inicialización no es costosa.** El bloque `static` es rápido, predecible y suficiente.

---

### 3. Criterio: Fuente Única de Verdad
* La palabra clave `static` en `Map<String, Supplier<ReportFormatFactory>>` ya asegura que exista una sola lista compartida en toda la aplicación. Un Singleton clásico nos obligaría a escribir código extra (método `getInstance()`, guardas de seguridad y sincronización) sin aportar nada nuevo, ya que la JVM garantiza por sí sola que la carga estática sea segura.
* **Conclusión del criterio:** **El `Map` estático ya es la fuente única de verdad.** Singleton solo agregaría código innecesario.

---

### 4. Criterio: Escenarios Futuros Razonables
* Sí. Si en el futuro el sistema evoluciona a un modelo donde varias universidades (*multi-tenant*) usen la misma aplicación pero cada una necesite formatos de reporte distintos, el patrón Singleton fallaría porque nos obligaría a compartir el mismo `Map` para todas. La estructura utilitaria actual es más flexible y permite evolucionar a registros independientes por institución si se requiere.
* **Conclusión del criterio:** **Existe un escenario futuro realista (*multi-tenant*) donde Singleton sería un obstáculo.** La solución utilitaria actual permite evolucionar sin trabas.


## Herramientas utilizadas
- Java 17, Apache Maven, VS Code, Git, GitHub


## Conclusiones
La lección más importante de este post-contenido no fue implementar patrones, 
sino **decidir cuál usar según el problema real**. En la Parte 1, elegir 
Abstract Factory sobre Factory Method no fue aleatorio: fue basado en que 
el sistema crea **familias coherentes** de productos, no productos aislados. 
Builder en la Parte 2 tampoco es "obligatorio" para muchos parámetros, es la herramienta correcta cuando necesitas validación atómica y garantizar 
que el objeto nunca queda a medio configurar. Pero lo más sorprendente fue 
descubrir que ReportFactoryRegistry no necesita Singleton: un patrón famoso 
no siempre es la respuesta. A veces, una clase utilitaria estática es suficiente 
y más flexible. El verdadero aprendizaje fue: **primero entender el problema con 
criterios explícitos, luego elegir el patrón, y finalmente implementarlo** 
y no al revés.