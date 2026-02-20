# Informe de Configuración de DMZ con Cisco Packet Tracer

---

## 1. Objetivo del laboratorio

El objetivo de este laboratorio fue diseñar y configurar una arquitectura de red con una Zona Desmilitarizada (DMZ) utilizando Cisco Packet Tracer.

Se implementó:

- Segmentación de red en tres zonas: LAN interna, DMZ y red externa.
- Traducción de direcciones (NAT) para publicar el servidor de la DMZ.
- Listas de Control de Acceso (ACL) para filtrar tráfico.
- Verificación de conectividad asegurando que solo el tráfico legítimo estuviera permitido.

El propósito principal fue simular un entorno real de seguridad perimetral.

---

## 2. Topología implementada

La red fue segmentada en tres zonas diferenciadas:

- **LAN (Internal Network)**: red privada donde se encuentra el PC interno.
- **DMZ**: zona intermedia donde se ubica el servidor web.
- **External Network**: red externa que simula Internet.

### Dispositivos utilizados

- 1 Router Cisco ISR (Router_FW)
- 3 Switches (uno por cada red)
- 1 PC_Internal
- 1 Server_DMZ
- 1 PC_External

### Interfaces del router

- GigabitEthernet0/0 → LAN
- GigabitEthernet0/1 → DMZ
- GigabitEthernet0/2 → Red Externa

La DMZ permite exponer servicios (HTTP) sin comprometer directamente la red interna.

---

## 3. Plan de direccionamiento IP

| Dispositivo               | IP            | Máscara           | Gateway        |
|---------------------------|---------------|-------------------|---------------|
| PC_Internal               | 192.168.1.10  | 255.255.255.0     | 192.168.1.1   |
| Server_DMZ                | 192.168.2.10  | 255.255.255.0     | 192.168.2.1   |
| PC_External               | 192.168.3.10  | 255.255.255.0     | 192.168.3.1   |
| Router_FW Gi0/0 (LAN)     | 192.168.1.1   | 255.255.255.0     | —             |
| Router_FW Gi0/1 (DMZ)     | 192.168.2.1   | 255.255.255.0     | —             |
| Router_FW Gi0/2 (Ext)     | 192.168.3.1   | 255.255.255.0     | —             |

---

## 4. Configuración aplicada

### 4.1 Configuración de interfaces

```bash
interface GigabitEthernet0/0
ip address 192.168.1.1 255.255.255.0
no shutdown

interface GigabitEthernet0/1
ip address 192.168.2.1 255.255.255.0
no shutdown

interface GigabitEthernet0/2
ip address 192.168.3.1 255.255.255.0
no shutdown

### 4.2 Configuración NAT

Se configuró NAT estático para publicar el servidor ubicado en la DMZ hacia la red externa.

```bash
ip nat inside source static 192.168.2.10 192.168.3.1

### 4.3 Configuración de ACL

Con el objetivo de reforzar la seguridad entre las distintas zonas de la red, se implementaron Listas de Control de Acceso (ACL) extendidas en el router.

La finalidad principal fue:

- Permitir únicamente el tráfico HTTP desde la red externa hacia el servidor publicado en la DMZ.
- Bloquear cualquier intento de comunicación iniciado desde la DMZ hacia la red interna (LAN).
- Mantener el aislamiento entre zonas sin afectar el tráfico legítimo.

---

#### ACL para permitir solo HTTP desde la red externa

Se configuró una ACL extendida aplicada en la interfaz conectada a la red externa:

```bash
access-list 100 permit tcp any host 192.168.3.1 eq 80
access-list 100 deny ip any any

Esta ACL:

Permite únicamente tráfico TCP hacia el puerto 80 (HTTP) del servidor publicado.
Bloquea cualquier otro tipo de tráfico entrante desde la red externa.
Posteriormente fue aplicada en la interfaz correspondiente:

```bash
interface GigabitEthernet0/2
ip access-group 100 in

ACL para bloquear tráfico DMZ → LAN

Para impedir que los dispositivos de la DMZ iniciaran comunicaciones hacia la red interna, se configuró la siguiente ACL:

```bash
access-list 110 permit tcp 192.168.2.0 0.0.0.255 192.168.1.0 0.0.0.255 established
access-list 110 deny ip 192.168.2.0 0.0.0.255 192.168.1.0 0.0.0.255
access-list 110 permit ip any any

Esta configuración realiza lo siguiente:

Permite únicamente tráfico TCP de respuesta (establecido) desde la DMZ hacia la LAN.
Bloquea cualquier intento nuevo de comunicación desde la DMZ hacia la red interna.
Permite el resto del tráfico necesario para no interrumpir la conectividad general.

La ACL fue aplicada en la interfaz conectada a la DMZ:

```bash
interface GigabitEthernet0/1
ip access-group 110 in

Consideraciones importantes

Las ACL se procesan de arriba hacia abajo.
El orden de las reglas es crítico.
Una regla mal posicionada puede bloquear tráfico legítimo o permitir accesos no deseados.
El uso de la palabra clave established permite que las respuestas a conexiones iniciadas desde la LAN no sean bloqueadas.

Con esta configuración se logró una segmentación efectiva, permitiendo acceso controlado al servidor web sin comprometer la seguridad de la red interna.

## 5. Verificaciones realizadas

Una vez finalizada la configuración de interfaces, NAT y ACLs, se realizaron diversas pruebas para comprobar el correcto funcionamiento de la red y validar que las políticas de seguridad se comportaban según lo previsto.

Las pruebas se llevaron a cabo desde los distintos dispositivos del laboratorio.

---

### 5.1 Verificación de conectividad básica

Antes de aplicar las ACL, se comprobó la conectividad entre los dispositivos y sus respectivas puertas de enlace:

- Ping desde **PC_Internal** hacia 192.168.1.1: Correcto.
- Ping desde **Server_DMZ** hacia 192.168.2.1: Correcto.
- Ping desde **PC_External** hacia 192.168.3.1: Correcto.

Estas pruebas confirmaron que el direccionamiento IP y la configuración de interfaces del router estaban correctamente implementados.

---

### 5.2 Verificación del funcionamiento de NAT

Se validó que el servidor ubicado en la DMZ fuera accesible desde la red externa utilizando la dirección pública configurada mediante NAT.

Desde **PC_External**:

- Acceso vía navegador a `http://192.168.3.1`: Correcto.

El servidor respondió correctamente, confirmando que el mapeo NAT estático estaba funcionando como se esperaba.

Asimismo, desde **PC_Internal** se verificó el acceso directo al servidor mediante su IP privada:

- Acceso a `http://192.168.2.10`: Correcto.

---

### 5.3 Verificación de las ACL

Una vez aplicadas las listas de control de acceso, se realizaron pruebas específicas para comprobar que las restricciones estaban activas:

- Desde **PC_External**, intento de `ping` a 192.168.3.1: Bloqueado correctamente.
- Desde **Server_DMZ**, intento de `ping` a 192.168.1.10 (LAN): Bloqueado correctamente.
- Desde **PC_Internal**, acceso HTTP al servidor en la DMZ: Permitido.

Estas pruebas demostraron que:

- Solo el tráfico HTTP desde la red externa estaba permitido.
- La DMZ no podía iniciar comunicaciones hacia la LAN.
- La red interna mantenía acceso legítimo al servidor.

---

### 5.4 Validación final

Finalmente, se utilizó la herramienta **Check Results** de Cisco Packet Tracer para validar automáticamente la actividad.

El resultado fue:

- Actividad completada correctamente (100%).

Esto confirma que la configuración cumple con todos los requisitos establecidos en el laboratorio.

## 6. Conclusiones

El desarrollo de este laboratorio permitió aplicar de forma práctica conceptos fundamentales de seguridad en redes y segmentación de infraestructura.

La implementación de una arquitectura con DMZ demostró la importancia de separar los servicios públicos de la red interna, reduciendo así la superficie de ataque y minimizando el impacto de posibles vulnerabilidades en servidores expuestos.

Durante la configuración se trabajó con tres elementos clave:

- Segmentación de red mediante distintas zonas (LAN, DMZ y red externa).
- Traducción de direcciones mediante NAT estático para publicar el servidor web.
- Listas de Control de Acceso (ACL) extendidas para restringir el tráfico entre zonas.

Uno de los aprendizajes más relevantes fue comprender que:

- Las ACL se procesan en orden secuencial.
- Una regla mal ubicada puede bloquear tráfico legítimo.
- Es imprescindible verificar conectividad básica antes de aplicar reglas de filtrado.
- El uso adecuado de palabras clave como `established` permite mantener sesiones válidas sin comprometer la seguridad.

Asimismo, se comprobó que la combinación de NAT y ACL permite exponer servicios de forma controlada, manteniendo protegida la red interna frente a accesos no autorizados.

En conclusión, la arquitectura implementada cumple con el objetivo del laboratorio: permitir acceso público al servidor web ubicado en la DMZ sin comprometer la seguridad de la red interna, aplicando principios básicos de defensa en profundidad.

## 7. Evidencias

Para respaldar la correcta implementación del laboratorio, se adjuntan capturas de pantalla en la carpeta `evidencias/` del repositorio.

Estas evidencias permiten verificar visualmente la configuración realizada y los resultados obtenidos.

---

### 7.1 Topología final

Captura de la topología completa en Cisco Packet Tracer donde se visualizan:

- Router_FW
- Switches de cada zona
- PC_Internal
- Server_DMZ
- PC_External
- Conexiones entre LAN, DMZ y red externa

Esta imagen demuestra la correcta segmentación de la red.

---

### 7.2 Configuración y estado del router

Capturas de:

- `show ip interface brief`
- `show ip nat translations`
- `show access-lists`

Estas evidencias confirman:

- Interfaces activas y correctamente direccionadas.
- NAT estático funcionando.
- ACL aplicadas en las interfaces correctas.

---

### 7.3 Pruebas de conectividad

Se incluyen capturas de:

- Ping exitoso desde PC_Internal hacia su gateway.
- Acceso HTTP exitoso desde PC_External al servidor publicado.
- Ping bloqueado desde la DMZ hacia la LAN.
- Ping bloqueado desde la red externa cuando no está permitido.

Estas pruebas validan el correcto funcionamiento de la segmentación y las políticas de filtrado.

---

### 7.4 Validación final

Captura de la herramienta **Check Results** mostrando la actividad completada al 100%.

Esta evidencia confirma que todos los requisitos del laboratorio fueron cumplidos correctamente.
