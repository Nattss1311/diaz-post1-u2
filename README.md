# diaz-post1-u2
Post-contenido — Exportación de reportes académicos con patrones creacionales justificados
# Decisión de diseño requerida — Parte 1 (1 de 2): ¿qué patrón aplica?

**Patrón elegido:** Abstract Factory

##Análisis mediante Preguntas Diagnósticas

| Pregunta Diagnóstica | Análisis y Respuesta para el Sistema de Reportes | Patrón al que apunta |
| :--- | :--- | :--- |
| **1. ¿El sistema crea un único tipo de producto que varía por formato, o varios productos relacionados que deben mantenerse coherentes entre sí dentro del mismo formato?** | El sistema crea **dos productos relacionados**: el cuerpo (tabla de notas) y el encabezado/pie de página. Ambos pertenecen a un mismo formato y deben mantenerse coherentes entre sí para no dañar el documento. | **Abstract Factory** (Familia completa de productos) |
| **2. Al agregar el futuro formato CSV, ¿basta con agregar una nueva implementación de un único producto, o hay que agregar una familia completa de piezas relacionadas?** | Se requiere agregar una **familia completa de piezas relacionadas**: una versión en CSV para el cuerpo y otra versión en CSV para el encabezado/pie de página. | **Abstract Factory** (Familia nueva completa) |
| **3. ¿El riesgo real del problema es "se instancia la clase equivocada" o es "se mezclan piezas de familias distintas y el documento queda inconsistente"?** | El riesgo real del problema es **mezclar piezas de familias o formatos distintos** (por ejemplo, combinar un cuerpo en Excel con un encabezado en PDF), lo cual generaría un documento inconsistente e inválido. | **Abstract Factory** (Riesgo de mezclar familias) |

#### Justificación y Descarte de la Alternativa

* **Por qué se eligió Abstract Factory:** Garantiza por diseño que todas las piezas que conforman el reporte (cuerpo, encabezado y pie de página) se creen juntas bajo un mismo formato compatible. Al centralizar la creación en una fábrica por formato, se imposibilita estructuralmente que el sistema combine un cuerpo de un tipo con un encabezado de otro.
* **Por qué se descartó Factory Method:** Factory Method está pensado para crear **un solo producto aislado**. Si usáramos Factory Method, tendríamos un creador separado para el cuerpo y otro creador separado para el encabezado, dejando en el código principal la responsabilidad de no cruzarlos. Esto mantendría abierto el riesgo de mezclar formatos distintos, fallando en resolver el problema central del ejercicio.

# Decisión de diseño requerida — Parte 1 (2 de 2): ¿cómo se resuelve la fábrica concreta en tiempo de ejecución?

**Mecanismo Elegido:** Registro Dinámico con `Map<String, Supplier<ReportFormatFactory>>` (Lookup Table)

#### Justificación de Diseño y Cumplimiento del Principio OCP

1. **Eliminación de Control de Flujo Rígido (`if/else` o `switch`):**
   - En lugar de evaluar condicionales rígidos sobre la cadena de texto con el nombre del formato, se utiliza un diccionario (`Map`) que asocia cada identificador de formato (`"pdf"`, `"excel"`, `"html"`) con un proveedor funcional (`Supplier<ReportFormatFactory>`).

2. **Extensibilidad sin Modificación (Open/Closed Principle - OCP):**
   - Cuando se agregue el formato **CSV** en el futuro, no será necesario modificar el método de resolución ni tocar el código cliente existente. Bastará con registrar la nueva fábrica concreta (`CsvReportFactory`) en el mapa de registro (mediante un bloque estático o un método `registerFactory`).

3. **Coherencia con Abstract Factory:**
   - Este mecanismo no reemplaza el patrón Abstract Factory; al contrario, actúa como un localizador/resolvedor de fábricas que retorna la instancia de la fábrica concreta adecuada (`ReportFormatFactory`). Una vez obtenida la fábrica desde el `Map`, el cliente la utiliza para crear la familia completa de productos (`ReportBody` y `ReportHeaderFooter`) garantizando la consistencia del documento.