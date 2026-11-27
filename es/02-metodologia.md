# Metodología detallada

Este documento desglosa los comandos y el razonamiento de cada paso del análisis. Todos los ejemplos usan direcciones IP y MAC **de ejemplo** (no las reales de la red auditada). Para el relato completo de hallazgos, ver [`01-informe-auditoria.md`](./01-informe-auditoria.md).

## Entorno de trabajo

- Linux (host), sin máquina virtual — ARP spoofing necesita acceso crudo al adaptador Wi-Fi, y el modo "bridged" de una VM sobre Wi-Fi es poco confiable por limitaciones del propio estándar 802.11.
- Wireshark con `dumpcap` usando *capabilities* de Linux (`cap_net_admin`, `cap_net_raw`) en vez de ejecutar todo con `sudo`. Principio de mínimo privilegio: un binario pequeño y auditado tiene privilegios elevados, mientras que los disectores de protocolo (que procesan datos no confiables de la red) corren sin ellos.
- Nmap, TShark, `arpspoof` (del paquete `dsniff`), Python 3 con el módulo estándar `socket`.

## 1. Descubrimiento de activos

```bash
sudo nmap -sn 192.168.1.0/24
```

`-sn` es un escaneo de "ping" — no toca puertos, solo confirma qué hosts están activos. Es el equivalente a "pasar lista": no debería generar ninguna interferencia perceptible en la red.

**Clasificación por MAC, no por IP.** Con DHCP, la IP de un dispositivo puede cambiar entre reinicios o renovaciones de arrendamiento. La MAC (en ausencia de randomización) es estable. Para distinguir una MAC de fábrica real de una generada aleatoriamente por software (común en celulares modernos por privacidad), se calcula el **bit universal/local (U/L)**: el segundo bit menos significativo del primer octeto de la MAC.

```
Primer octeto en binario: xxxxxx1x  →  bit U/L = 1  →  MAC generada por software (aleatoria)
Primer octeto en binario: xxxxxx0x  →  bit U/L = 0  →  MAC de fábrica (asignada por el fabricante, OUI real)
```

Ejemplo (ficticio): `3E:A1:9C:40:12:7B` → primer octeto `3E` = `00111110` en binario → segundo bit menos significativo = `1` → MAC aleatoria.

## 2. Fingerprinting de puertos

```bash
sudo nmap -sS -sV -p- 192.168.1.50
```

- `-sS`: SYN scan — envía solo el primer paquete del *three-way handshake* (SYN) y nunca completa la conexión. Más rápido a gran escala y deja menos rastro en los logs del objetivo que un escaneo de conexión completa.
- `-sV`: intenta identificar la versión/servicio detrás de cada puerto abierto (banner grabbing + base de huellas conocidas).
- `-p-`: escanea los 65 535 puertos, no solo los "comunes".

**Nota importante sobre falsos positivos:** Nmap identifica servicios por coincidencia de bytes contra una base de huellas, no por certeza semántica. En este proyecto, dos fingerprints (un protocolo de monitoreo de servidores y un motor de videojuego de estrategia) resultaron ser coincidencias casuales de bytes sin relación real con la función de los dispositivos. Regla práctica: si el fingerprint no encaja con el *rol* plausible del dispositivo (ej. un cliente IoT reportando como si fuera un servidor de monitoreo), tratarlo como sospechoso hasta verificar.

Para servicios desconocidos, el siguiente paso es inspección directa:

```bash
echo -n "<frase de consulta>" | nc -u -w2 192.168.1.50 <puerto>
```

## 3. ARP spoofing (interceptación activa autorizada)

> Requiere autorización explícita de los dueños de los dispositivos interceptados. Ver [`alcance-y-autorizacion.md`](./alcance-y-autorizacion.md).

**Por qué hace falta:** en un switch (a diferencia de un hub), cada puerto solo recibe el tráfico destinado a su propia MAC, más broadcast/multicast. Un equipo conectado al mismo switch que un celular y un dispositivo IoT **no ve** la conversación unicast entre ambos, salvo que se ponga deliberadamente en medio.

**Mecanismo:** ARP no tiene autenticación — cualquier equipo puede anunciar "yo soy tal IP" y los demás lo creen sin verificar. El ataque consiste en:

1. Activar el reenvío de paquetes del kernel, para no cortar la conectividad real de las víctimas (y evitar un DoS accidental):
   ```bash
   sudo sysctl -w net.ipv4.ip_forward=1
   ```
2. Envenenar la caché ARP de la víctima A, haciéndose pasar por la víctima B (en una terminal):
   ```bash
   sudo arpspoof -i <interfaz> -t <IP_víctima_A> <IP_víctima_B>
   ```
3. Envenenar la caché ARP de la víctima B, haciéndose pasar por la víctima A (en otra terminal, simultánea):
   ```bash
   sudo arpspoof -i <interfaz> -t <IP_víctima_B> <IP_víctima_A>
   ```
4. Capturar con Wireshark en paralelo, filtrando por la IP de interés:
   ```
   ip.addr == <IP_objetivo>
   ```

**Las dos instancias de `arpspoof` son ataques independientes, uno por cada sentido de la conversación.** Si una de las dos tiene un error (de tipeo, por ejemplo), se pierde solo esa mitad del tráfico, no toda la intercepción — un buen diagnóstico diferencial si la captura muestra tráfico unidireccional.

### Problemas de entorno que pueden simular un "fallo del ataque"

1. **Firewall del propio equipo atacante.** Un firewall de host activo por defecto (ej. `ufw` en distribuciones basadas en Debian/Ubuntu) suele traer la cadena `FORWARD` de `iptables` en política `DROP`. Esto bloquea en silencio el reenvío de paquetes de terceros a través del equipo atacante, **sin generar ningún error visible** — parece que el ataque "no funciona" o que la víctima "perdió internet", cuando en realidad el equipo atacante nunca reenvía lo que intercepta.

   Diagnóstico:
   ```bash
   sudo iptables -L FORWARD -n -v
   ```
   Si la política es `DROP` y hay paquetes contados ahí, esa es la causa. Corrección específica (sin desactivar el firewall completo), para la herramienta `ufw`:
   ```bash
   sudo ufw route allow in on <interfaz> out on <interfaz>
   ```

2. **ICMP Redirect.** Cuando un equipo reenvía un paquete de vuelta por la misma interfaz por la que llegó (exactamente lo que hace un ataque MITM), el kernel de Linux genera automáticamente un mensaje ICMP Redirect hacia el emisor original, sugiriéndole "habla directo, no pases por mí". Si la víctima le hace caso, el ataque se rompe silenciosamente. Mitigación:
   ```bash
   sudo sysctl -w net.ipv4.conf.all.send_redirects=0
   sudo sysctl -w net.ipv4.conf.<interfaz>.send_redirects=0
   ```

3. **Restauración incorrecta al detener el ataque.** Al interrumpir `arpspoof`, la herramienta intenta restaurar la caché ARP real de las víctimas. En algunos escenarios de error previo puede restaurar apuntando a una MAC de todos ceros en vez de la MAC verdadera, dejando temporalmente sin conectividad al dispositivo. Reiniciar limpiamente el proceso suele resolverlo.

**Checklist de diagnóstico cuando el MITM "no funciona":**
- ¿`ip_forward` está en 1?
- ¿Las dos instancias de `arpspoof` tienen las IPs correctas (sin errores de tipeo)?
- ¿La cadena `FORWARD` de `iptables` deja pasar el tráfico (no solo ICMP)?
- ¿Se desactivaron los ICMP Redirect salientes?
- ¿El problema afecta ambos sentidos del tráfico por igual, o solo uno? (un problema que afecta solo un sentido casi nunca es "la víctima ganando una carrera de ARP" — primero descartar el propio firewall)

## 4. Ingeniería inversa de un protocolo binario desconocido

Proceso general reutilizable, aplicado al canal de control del dispositivo IoT de este proyecto:

1. **Capturar múltiples muestras** del mismo tipo de mensaje, de tamaños distintos si es posible.
2. **Buscar bytes constantes** al inicio de todos los mensajes → candidato a "magic number" (identificador de protocolo), no a credencial.
3. **Buscar campos que cambian de forma predecible** entre mensajes consecutivos (ej. un contador que sube de 1 en 1) → generalmente metadatos de secuencia, no autenticación.
4. **Aislar el campo de longitud**, si existe, comparando mensajes de distinto tamaño de carga útil.
5. **Para el último byte o bloque "raro":** probar primero la hipótesis más simple antes que una compleja. Para un byte de verificación, orden sugerido de hipótesis:
   - XOR de todos los bytes anteriores.
   - Suma de todos los bytes anteriores, módulo 256 (checksum aditivo simple).
   - Solo si ninguna de las anteriores cuadra con varias muestras independientes, considerar un CRC estándar o un HMAC (en cuyo caso el campo *sí* sería una medida de seguridad real, no cosmética).
6. **Confirmar con al menos 3–4 muestras independientes** antes de dar una fórmula por buena.
7. **Experimento controlado para identificar el significado de un campo de datos:** cambiar una sola variable a la vez (ej. una sola acción del dispositivo por captura) y comparar qué bytes aparecen únicamente cuando esa acción ocurrió.
8. **Validar con una prueba de concepto activa:** construir un mensaje a mano con los campos reconstruidos y enviarlo directo al dispositivo. Si el dispositivo responde/actúa como se esperaba, el modelo del protocolo queda confirmado, no solo es una hipótesis sobre bytes.
9. **Probar los límites del protocolo, no solo el camino feliz:** por ejemplo, enviar el mismo comando con un campo de secuencia/contador fuera de la secuencia esperada, para verificar si el receptor valida ese campo o lo ignora (vulnerabilidad de repetición/replay si lo ignora).

### Ejemplo de prueba de concepto (estructura genérica, sin los valores reales confirmados)

```python
import socket

sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
sock.connect(("<IP_del_dispositivo>", "<puerto>"))

# payload construido a mano según el formato de trama reconstruido:
# MAGIC(4) + TIPO(2) + CONTADOR(2) + LARGO(2) + DATOS(LARGO) + CHECKSUM(1)
payload = bytes.fromhex("<trama reconstruida — valores reales omitidos de esta publicación>")
sock.send(payload)

response = sock.recv(1024)
print(response.hex())
sock.close()
```

Ver [`exploit/README.md`](../exploit/README.md) para la explicación completa de qué partes del código real se omitieron de este repositorio y por qué.

## 5. Análisis de metadatos en tráfico cifrado

Sin romper ningún cifrado, hay tres fuentes de metadatos típicamente visibles aun cuando el contenido viaja por HTTPS/TLS:

| Filtro de Wireshark | Qué revela |
|---|---|
| `dns` | Dominios resueltos en texto plano (antes de que exista la conexión cifrada) |
| `tls.handshake.type == 1` | El campo SNI (*Server Name Indication*) del Client Hello — el primer paquete de cualquier conexión TLS, que indica a qué dominio se está conectando, sin cifrar |
| Estadísticas → Conversaciones | Volumen de datos y duración de cada conexión — sugiere el tipo de actividad (ej. mucho volumen sostenido = probable video) sin revelar el contenido específico |

Complementario: Estadísticas → Jerarquía de protocolos da una vista agregada de qué protocolos dominan una captura completa — útil para detectar anomalías (protocolos industriales o de videojuegos antiguos apareciendo en un celular normal son casi siempre falsos positivos del disector, no hallazgos reales; se confirman o descartan siguiendo el flujo correspondiente con "Seguir secuencia").
