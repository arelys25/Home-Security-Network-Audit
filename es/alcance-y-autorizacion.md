# Alcance y autorización

Este proyecto se realizó sobre una red doméstica real, con **autorización previa, explícita y por escrito** de los integrantes de la familia dueños de los dispositivos analizados, firmada antes de iniciar cualquier captura o interceptación activa. El documento de consentimiento original (en español, con nombres y fechas reales de la familia) no se publica en este repositorio por privacidad; a continuación se resume su contenido.

## Qué se autorizó

- Escaneo de descubrimiento de la red (identificar qué dispositivos están conectados).
- Captura de tráfico con Wireshark para analizar qué tipo de datos circulan.
- **ARP spoofing**: una técnica que redirige temporalmente el tráfico de un dispositivo a través del equipo de la auditora antes de que siga su camino normal, para poder observarlo. Se advirtió explícitamente que podía notarse el internet un poco más lento durante esos momentos, sin que debiera cortarse por completo.

## Qué sí se podía ver, y qué no (explicado a la familia en términos no técnicos)

La mayoría del tráfico de internet actual (redes sociales, streaming, banca, buscadores) usa cifrado HTTPS — el equivalente a una carta dentro de un sobre sellado a prueba de manipulación: aunque la carta pase físicamente por las manos de quien intercepta, no se puede abrir el sobre y leer el contenido.

**Sí se podía ver, incluso en apps que cifran su contenido:**
- A qué sitios o apps se conectó un dispositivo (el dominio, no la página o publicación específica).
- Aproximadamente cuánto tiempo estuvo activa una conexión y cuántos datos transfirió.

**No se podía ver en ningún caso:**
- El contenido específico visto, publicado o buscado.
- El contenido de mensajes privados.
- Contraseñas de sitios con HTTPS.

Es el equivalente a ver un recibo telefónico (a quién se llamó y cuánto duró la llamada) en vez de escuchar la llamada.

Donde sí se podía encontrar información completa y sin cifrar era en dispositivos con seguridad débil por diseño — cámaras IP, focos o enchufes inteligentes, impresoras — que en ocasiones no cifran nada de lo que envían. Ese fue, de hecho, uno de los objetivos reales del proyecto: detectar si había en casa algún dispositivo de este tipo exponiendo información sin que la familia lo supiera.

## Compromisos explícitos de la auditora

- No guardar, compartir ni usar contraseñas o datos personales que aparecieran en las capturas. Cualquier dato sensible se oculta o elimina antes de aparecer en cualquier reporte.
- No modificar, bloquear ni dañar ningún dispositivo.
- Nada de lo capturado sale de la red de casa ni se comparte fuera del proyecto — lo que se publica de forma pública (este repositorio) pasó por un proceso adicional de redacción, descrito en el README principal y en [`exploit/README.md`](../exploit/README.md).

## Periodo y posibilidad de exclusión

El trabajo de campo se realizó en sesiones puntuales durante julio y agosto de 2026, avisadas con anticipación a la familia. Cualquier integrante tenía la opción de excluir su dispositivo del análisis sin ninguna consecuencia; esa opción se mantuvo abierta durante todo el proyecto.

## Nota sobre esta publicación

Este repositorio es una versión **sanitizada** del trabajo original, pensada para portafolio académico y divulgación de la metodología. Se eliminaron o sustituyeron por marcadores genéricos: direcciones IP y MAC reales, el nombre de la red Wi-Fi (SSID), la marca y modelo exacto del dispositivo IoT vulnerable, y cualquier dato que pudiera identificar a los miembros de la familia o sus hábitos de uso específicos (más allá de lo estrictamente necesario para explicar un hallazgo, como "un familiar" en vez de un nombre). Ver también la nota de divulgación responsable en el README principal respecto al dispositivo IoT vulnerable.
