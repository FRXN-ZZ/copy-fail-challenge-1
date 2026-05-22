# Reporte Técnico: Análisis del Exploit Copy-Fail (CVE-2026-31431)

Este reporte detalla los fundamentos técnicos subyacentes en la vulnerabilidad de corrupción de memoria lógica conocida como "Copy-Fail", analizando desde el origen del defecto en el subsistema criptográfico hasta su impacto directo en las estructuras de control de acceso del sistema operativo.

---

### 1. Bug Raíz: Ubicación y Mecánica Teórica

El defecto principal reside en el subsistema criptográfico del Kernel de Linux, específicamente en el archivo `crypto/algif_aead.c` dentro de la función interna encargada de procesar la recepción de mensajes cifrados: `_aead_recvmsg()`. 

El origen del problema data de una optimización de rendimiento introducida en el año 2017. Con el fin de evitar la duplicación innecesaria de búferes en memoria y acelerar las operaciones criptográficas simétricas, se implementó un mecanismo de procesamiento *in-place*. Al realizar esto, el kernel enlazaba las páginas de datos a través de la función `sg_chain()`, provocando que la lista de dispersión/recolección de origen (TX SGL) y la de destino (RX SGL) apuntaran al mismo bloque de memoria física compartida. La asignación errónea ocurre al invocar `aead_request_set_crypt()` pasando punteros de origen idénticos a los de destino (`src == dst`), eliminando el aislamiento lógico entre operaciones de lectura y escritura.

---

### 2. El Peligro del Desbordamiento Lógico en el Tag

Durante una operación de descifrado AEAD (Autenticación con Datos Asociados), el algoritmo necesita escribir y validar un tag de autenticación al final de la secuencia de datos procesados, ubicado en la posición matemática `dst[assoclen + cryptlen]`. 

Bajo condiciones normales de diseño operativo, este almacenamiento ocurre en un búfer volátil o aislado del espacio de usuario. Sin embargo, debido al bug raíz donde la lista de destino comparte referencias directamente con el subsistema de paginación activa, la llamada criptográfica escribe esos bytes del tag de manera forzada sobre el almacenamiento intermedio del núcleo. Esto rompe la atomicidad del aislamiento de memoria, permitiendo que un usuario no privilegiado altere bytes adyacentes controlados indirectamente mediante llamadas al sistema controladas como `splice()`.

---

### 3. Naturaleza Sigilosa ("Stealthy") del Vector de Ataque

Una de las propiedades críticas de este exploit es su capacidad para evadir los mecanismos tradicionales de auditoría forense y de integridad de archivos. El ataque se califica como "stealthy" (sigiloso) porque **no modifica un solo byte del archivo binario almacenado de manera física en el disco duro**.

El exploit manipula exclusivamente las copias volátiles que residen en la memoria RAM del sistema. Al interactuar con los descriptores de archivos, el kernel lee el binario objetivo una vez y lo mantiene en memoria de acceso rápido para optimizar la ejecución. La corrupción lógica altera esta representación en memoria; por ende, las sumas de verificación del disco (como `sha256sum /usr/bin/su`) permanecen intactas. Un reinicio del sistema o el vaciado de los búferes volátiles elimina cualquier rastro de la alteración, volviendo el ataque invisible ante herramientas estáticas de detección de intrusos.

---

### 4. Conexión Académica: Page Cache, Inodos y Elevación de Privilegios

El exploit entrelaza de forma avanzada múltiples conceptos de administración de sistemas de archivos y memoria del kernel vistos en clase:

* **Page Cache:** Es el subsistema central explotado. Al usar `splice()`, el exploit asocia las páginas de memoria de la API de sockets criptográficos directamente con las páginas del *Page Cache* que almacenan el ejecutable `/usr/bin/su`.
* **Inodos:** El inodo mapea los metadatos fijos en el disco. El exploit no altera el inodo ni sus bloques de datos en el dispositivo de bloques, burlando restricciones de solo lectura a nivel físico.
* **Setuid y Chmod:** El binario `/usr/bin/su` posee el bit *Setuid* activo (configurado originalmente mediante `chmod +s`). Cuando un usuario ejecuta este binario, el kernel inicia el proceso bajo el contexto de identidad del propietario del archivo (que es `uid=0`, o `root`). Como los 4 bytes críticos inyectados en el *Page Cache* alteran la lógica interna de validación de contraseñas de `su` en la memoria RAM, el binario modificado concede una consola de comandos con los privilegios heredados del dueño del inodo, transformando la identidad del proceso de `student` a `root` de forma instantánea.

---
### 5. Lección Aprendida: Paradoja de Cambios Razonables

Este caso de estudio proporciona una lección invaluable sobre la ingeniería de software segura: la acumulación de decisiones de diseño lógicas y aisladas puede resultar en una catástrofe de seguridad combinatoria.

Por un lado, la optimización *in-place* de 2017 era completamente razonable, ya que buscaba ahorrar ciclos de reloj y uso de memoria RAM. Por otro lado, los mecanismos de optimización de transferencia de datos de `splice()` e interfaces como `AF_ALG` son herramientas de sistema estándar diseñadas para maximizar la velocidad de aplicaciones en espacio de usuario. Cada subsistema funcionaba de manera impecable bajo sus propias especificaciones de diseño. El fallo catastrófico nace en la **intersección inesperada de subsistemas**, demostrando que la seguridad no puede validarse únicamente de forma modular, sino que requiere un análisis holístico de los efectos colaterales cuando diferentes APIs de bajo nivel comparten recursos críticos de la memoria del núcleo.
