# **ADR-001: Refactorización del módulo de autenticación debido a vulnerabilidades críticas de seguridad y problemas de mantenibilidad**

## **Contexto**

Durante la auditoría técnica realizada en la **Fase 3**, se evaluó el módulo de autenticación actual, el cual implementa funcionalidades básicas de login y registro mediante endpoints REST en Spring Boot.  
El análisis evidenció múltiples fallas graves que comprometen tanto la **seguridad** de la aplicación como su **mantenibilidad**. Entre los problemas identificados se encuentran:

*   Exposición del *hash* MD5 de la contraseña en las respuestas del login.
*   Uso de MD5 sin *salt*, técnica considerada obsoleta e insegura para almacenar contraseñas.
*   Vulnerabilidad a **SQL Injection** debido al uso de `Statement` y concatenación de strings.
*   Validaciones insuficientes en el proceso de registro.
*   Manejo incorrecto de códigos HTTP (retorno de 200 OK incluso en fallos).
*   Modelo `User` con campos públicos y ausencia de encapsulación.
*   Logging de información sensible.
*   Dependencia directa de `AuthService` hacia una implementación concreta de repositorio.

Estas deficiencias representan riesgos severos, tales como robo de credenciales, alteración de datos, suplantación de identidad y brechas de seguridad. Además, dificultan la extensión del sistema y aumentan la deuda técnica.

Dado el impacto potencial y la criticidad de estas vulnerabilidades, se hace necesario adoptar una decisión arquitectónica que permita corregir el módulo antes de avanzar con desarrollos futuros.

***

## **Decisión**

Se optó por realizar una **refactorización parcial estructurada** del módulo de autenticación, orientada por principios **SOLID**, **Clean Code** y buenas prácticas de seguridad.

### **1. Mitigar SQL Injection mediante PreparedStatement**

Se reemplazarán todos los `Statement` por `PreparedStatement` (o `JdbcTemplate`) para impedir que los parámetros del usuario alteren la estructura de las consultas SQL.  
Esta acción atiende directamente la falla crítica demostrada durante las pruebas funcionales.

### **2. Sustituir MD5 por BCrypt para almacenamiento seguro**

El sistema implementará `BCryptPasswordEncoder`, que incorpora salting automático y una función de hashing resistente a ataques de fuerza bruta.  
Esto elimina la vulnerabilidad asociada al uso de MD5 y evita la exposición de hashes predecibles.

### **3. Encapsular el modelo y utilizar DTOs con validaciones**

Los atributos de `User` pasarán a ser privados y se incorporarán DTOs para las operaciones de login y registro.  
Estos DTOs incluirán validaciones con `@Valid` y nombres explícitos, reemplazando parámetros ambiguos como `u`, `p`, y `e`.  
Este cambio mejora la claridad del código y reduce el riesgo de errores en la API.

### **4. Ajustar los códigos HTTP según estándares REST**

*   Login fallido: **401 Unauthorized**
*   Errores de validación: **400 Bad Request**
*   Operaciones exitosas: **200 / 201**

Esto evita el uso incorrecto de `200 OK` en escenarios de error.

### **5. Introducir una interfaz de repositorio (DIP)**

Se define `IUserRepository` para desacoplar `AuthService` de una implementación concreta.  
El cambio facilita pruebas unitarias, permite sustituir el backend de persistencia y mejora la flexibilidad del diseño.

***

## **Consecuencias**

### **Consecuencias positivas**

*   Eliminación de la vulnerabilidad de SQL Injection.
*   Almacenamiento seguro de contraseñas mediante un algoritmo robusto (BCrypt).
*   Reducción de la exposición de información sensible en respuestas y logs.
*   Código más limpio, legible y mantenible gracias a DTOs, encapsulación y nombres adecuados.
*   Respuestas HTTP consistentes con los principios REST.
*   Arquitectura más modular, extensible y apta para pruebas gracias al desac acoplamiento introducido.

### **Consecuencias negativas o riesgos**

*   Requiere tiempo adicional de implementación y revisión.
*   Puede causar rupturas en integraciones existentes que dependan de las respuestas anteriores del API.
*   Se necesitará un proceso para migrar las contraseñas almacenadas en MD5 a BCrypt.
*   Incremento inicial en la complejidad de la arquitectura y dependencias del proyecto.

***

## **Alternativas consideradas**

### **1. Reescritura completa del módulo de autenticación**

Se descartó porque aumentaría significativamente el tiempo y riesgo del desarrollo.  
La base existente es refactorizable y no amerita una reconstrucción total para los objetivos actuales del proyecto.

### **2. Aplicar únicamente parches críticos**

Esta opción hubiera mitigado las vulnerabilidades de seguridad más obvias, pero dejaría intactos problemas graves de diseño: falta de encapsulación, parámetros ambiguos, estructura rígida, manejo deficiente de errores y respuestas HTTP, entre otros.  
Se descartó por riesgo de aumentar la deuda técnica.

### **3. Migrar inmediatamente a Spring Security con JWT**

Considerado, pero descartado para esta fase, ya que ampliaría considerablemente el alcance del proyecto.  
Podría ser retomado en una futura iteración como mejora evolutiva.
