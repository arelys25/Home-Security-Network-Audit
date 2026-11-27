# Mitigaciones y recomendaciones

## Rediseño del protocolo del dispositivo IoT vulnerable

Las dos vulnerabilidades confirmadas en el hallazgo central (ver [`01-informe-auditoria.md`](./01-informe-auditoria.md)) son:

1. **Ausencia total de autenticación** en el canal de control.
2. **Ausencia de protección contra repetición (replay)** — el campo de contador existe en el protocolo, pero el receptor no lo valida.

Un primer instinto razonable pero incorrecto sería "esconder" u ofuscar campos como el *magic number* o el checksum. **Esto no resuelve nada**: un atacante que ya capturó un paquete válido no necesita entender su estructura para reproducirlo tal cual — ocultar un campo no detiene una repetición ni una falsificación si el atacante solo copia bytes que ya vio funcionar. Es el error clásico de confundir **seguridad por oscuridad** con autenticación real.

La solución real combina tres piezas complementarias:

| Mecanismo | Qué reemplaza | Qué logra |
|---|---|---|
| **Llave secreta compartida (pre-shared key)**, establecida en el emparejamiento inicial entre el dispositivo y la app legítima, que nunca viaja en cada mensaje | Nada — hoy no existe | Da a cada dispositivo un secreto real que un observador pasivo de la red nunca ve |
| **HMAC** calculado con esa llave, sustituyendo el campo de checksum actual | El checksum aditivo simple (suma mod 256), que cualquiera puede calcular sin ningún secreto | Sin la llave, es computacionalmente inviable generar un HMAC válido para un mensaje nuevo o modificado — un paquete falso se detecta y se rechaza |
| **Validación de estado en el receptor**: el dispositivo recuerda el último valor de contador aceptado y rechaza cualquier paquete con un valor igual o menor | Nada — hoy el contador existe pero no se valida | Un paquete capturado una sola vez deja de ser reutilizable: el receptor, no el formato del mensaje, es quien impide la repetición |

Con las tres piezas juntas: un atacante que solo escucha la red (observación pasiva) no puede forjar comandos nuevos (no tiene la llave) ni reproducir uno viejo (el contador ya fue usado y el receptor lo recuerda).

## Recomendaciones adicionales (contexto general, más allá de este protocolo específico)

- **Segmentación de red para IoT.** Separar los dispositivos IoT del resto de la red doméstica mediante una VLAN o una red de invitados dedicada. Esto no arregla la vulnerabilidad del protocolo, pero limita drásticamente el radio de daño si un dispositivo IoT se ve comprometido: un atacante que ya controla el foco no debería poder, desde ahí, alcanzar directamente laptops, celulares o cámaras de la red principal.
- **Canal cifrado para el control local.** Usar TLS/DTLS (o un equivalente autenticado) para el canal de control LAN, en vez de un protocolo binario propietario en texto plano sobre TCP.
- **Revisar si existe ya un CVE público** para la familia de hardware/firmware involucrada. Este tipo de módulo de control WiFi barato y sin autenticación es un patrón conocido en el ecosistema IoT de bajo costo — vale la pena verificar si el hallazgo es nuevo o ya está documentado, antes de tratarlo como un descubrimiento aislado.
- **Actualizar/sustituir el dispositivo si el fabricante no ofrece una corrección.** Si el firmware no se actualiza de forma remota (común en esta categoría de producto), la mitigación práctica a nivel de usuario final es la segmentación de red mencionada arriba, no esperar un parche.

## Recomendaciones generales de higiene de red doméstica (de la auditoría completa)

- **Metadatos siguen siendo metadatos, aunque el contenido esté cifrado.** DNS y SNI de TLS revelan qué servicios se usan incluso cuando HTTPS protege el contenido. Si la privacidad frente a otros miembros de la red (o un ISP) es una preocupación, considerar DNS cifrado (DoH/DoT) — aunque esto no oculta el SNI por sí solo, salvo que también se use ECH (*Encrypted Client Hello*) cuando el servidor lo soporte.
- **No asumir que "filtered" significa "seguro".** Un host con todos los puertos filtrados puede simplemente tener un firewall que oculta el estado real de los servicios, no necesariamente ausencia de servicios vulnerables.
- **Cambiar las credenciales por defecto del panel de administración del router**, si todavía no se ha hecho — fuera del alcance de esta auditoría específica, pero una recomendación estándar que aplica a cualquier red doméstica.
