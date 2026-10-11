# Laboratorio de Seguridad Perimetral: FortiGate, DMZ y Segmentación de VLANs en GNS3

🎥 https://youtu.be/haCspUkcse4

<img width="931" height="547" alt="imagen" src="https://github.com/user-attachments/assets/a0a41763-f703-4e31-b179-4a4a56c41a34" />


---

## Índice
1. [Propósito del Laboratorio](#propósito-del-laboratorio)
2. [Topología de Red y Arquitectura](#topología-de-red-y-arquitectura)
3. [Tabla de Direccionamiento IP](#tabla-de-direccionamiento-ip)
4. [Diario de Configuración y Troubleshooting Operativo](#diario-de-configuración-y-troubleshooting-operativo)
5. [Configuraciones Implementadas Paso a Paso](#configuraciones-implementadas-paso-a-paso)
6. [Políticas de Seguridad (Firewall Policies)](#políticas-de-seguridad-firewall-policies)
7. [Validación y Pruebas Analíticas](#validación-y-pruebas-analíticas)
8. [Diagramas y Evidencias Visuales](#diagramas-y-evidencias-visuales)
9. [Running Configurations Completas](#running-configurations-completas)
10. [Scripts y Archivos Utilizados](#scripts-y-archivos-utilizados)
11. [Conclusión](#conclusión)

---

## Propósito del Laboratorio
El objetivo integral de esta práctica es diseñar, implementar, asegurar y diagnosticar una infraestructura corporativa simulada en GNS3. Se utiliza un appliance virtual **FortiGate (FortiOS)** como núcleo de seguridad (NGFW) y enrutamiento (Router-on-a-Stick). 

Los propósitos técnicos abarcan:
* **Segmentación L2/L3:** Aislamiento del tráfico interno mediante VLANs administradas por un Switch Cisco y enrutadas por subinterfaces en el FortiGate.
* **Implementación de DMZ:** Creación de una Zona Desmilitarizada físicamente aislada para servidores que requieren exposición pública restringida.
* **Egress Filtering Avanzado:** Restricción de salida a Internet para la DMZ basándose en objetos FQDN (`archive.ubuntu.com`), bloqueando IPs no autorizadas mediante políticas Zero Trust.
* **Source NAT (SNAT):** Traducción de direcciones privadas a públicas para proveer navegación segura a los usuarios de las VLANs.
* **Troubleshooting de Virtualización:** Resolución de conflictos de red entre GNS3, adaptadores Host-Only (VMnet1/VMnet8) y el Firewall de Windows, implementando la interfaz `eth1` para estabilizar el enlace WAN.

---

## Topología de Red y Arquitectura
La arquitectura divide la red en tres zonas lógicas de confianza:

1. **Zona WAN (Untrusted - Puerto 1):** Conectada al nodo Cloud mediante la interfaz `eth1` de la GNS3 VM. Provee IP dinámica, ruta por defecto nativa y salida a Internet evadiendo bloqueos del host físico.
2. **Zona LAN (Trusted - Puerto 3):** Segmentada mediante un Switch Cisco (SWA) configurado con enlaces troncales (802.1Q).
   * **VLAN 10 (LIMITADOS):** Red de empleados regulares (USER1).
   * **VLAN 20 (SSH):** Red de administración y control (USER2).
3. **Zona DMZ (Semi-Trusted - Puerto 2):** Red aislada para un Servidor Ubuntu de uso específico, con reglas de entrada/salida altamente restrictivas.

---

## Tabla de Direccionamiento IP basado en matricula (20240886)

| Dispositivo / Nodo | Interfaz / VLAN | Dirección IP / Prefijo | Gateway Predeterminado | DNS Asignado |
| :--- | :--- | :--- | :--- | :--- |
| **FortiGate** | `port1` (WAN) | DHCP (Dinámica por NAT) | Automático (GNS3 `eth1`) | Automático / `8.8.8.8` |
| **FortiGate** | `port2` (DMZ) | `10.20.24.1 / 28` | N/A | N/A |
| **FortiGate** | `port3.10` (VLAN 10) | `10.10.10.1 / 24` | N/A | N/A |
| **FortiGate** | `port3.20` (VLAN 20) | `10.20.20.1 / 24` | N/A | N/A |
| **Ubuntu Srv** | `eth0` (DMZ) | `10.20.24.2 / 28` | `10.20.24.1` | `8.8.8.8` |
| **USER1** | `eth0` (VLAN 10) | `10.10.10.X / 24` | `10.10.10.1` | `8.8.8.8` |
| **USER2** | `eth0` (VLAN 20) | `10.20.20.X / 24` | `10.20.20.1` | `8.8.8.8` |

---

## Diario de Configuración y Troubleshooting Operativo

Durante el desarrollo del laboratorio, se presentaron y resolvieron múltiples incidencias arquitectónicas:

1. **Incidencia de Acceso GUI (VMnet8 vs Nube NAT):**
   * *Problema:* El FortiGate recibió inicialmente una IP de una red aislada (`192.168.42.11`) inaccesible desde el host físico, y posteriormente una IP de VMnet8 (`192.168.184.133`) que sufría bloqueos de ping (Time Out).
   * *Solución Aplicada:* Se identificó que el Firewall de Windows estaba bloqueando el tráfico ICMP de la red "No Identificada" de VMware. Se desactivó el firewall para la red pública, permitiendo el acceso HTTPS a la interfaz gráfica.
2. **Incidencia de Salida a Internet (Pérdida de Ruta por Defecto):**
   * *Problema:* Aunque había acceso a la GUI, el FortiGate mostraba 100% de pérdida de paquetes hacia `8.8.8.8` y no poseía la ruta `S* 0.0.0.0/0`.
   * *Solución Aplicada:* Se reemplazó la conexión del nodo Cloud desde los adaptadores virtuales físicos hacia la interfaz `eth1` nativa de la máquina virtual de GNS3. Se ejecutó `execute interface dhcpclient-renew port1`, lo que estabilizó el enrutamiento WAN de forma definitiva.
3. **Restricción de Políticas Multi-Interfaz:**
   * *Problema:* Al intentar crear la regla de salida a Internet para las VLANs 10 y 20, la GUI del FortiGate no permitía seleccionar más de una interfaz de origen (Incoming Interface).
   * *Solución Aplicada:* Se habilitó globalmente la característica en FortiOS navegando a *System > Feature Visibility* y encendiendo el interruptor *Multiple Interface Policies*.

---

## Configuraciones Implementadas Paso a Paso

### 1. Configuración de Capa 2 (Switch Cisco)
Se crearon las VLANs de la base de datos, se asignaron los puertos de acceso para los usuarios y se configuró un enlace troncal dot1q hacia el firewall. Se guardó la configuración en la NVRAM para evitar pérdida de datos en GNS3.

### 2. Preparación del Enlace WAN (FortiGate CLI)
Se habilitó el protocolo DHCP en el `port1` y se abrieron los puertos de administración para permitir la configuración remota.
```text
config system interface
    edit "port1"
        set mode dhcp
        set allowaccess ping https http ssh fgfm
    next
end
```

### 3. Configuración de Subinterfaces (Router-on-a-Stick)
Desde la GUI (Network > Interfaces), se crearon subinterfaces lógicas colgando del `port3` físico, asignando los identificadores de VLAN correspondientes, IPs estáticas y habilitando PING para facilitar el diagnóstico interno.
<img width="1488" height="238" alt="imagen" src="https://github.com/user-attachments/assets/91f90c41-1cd0-46b1-a79a-ca8c0341a179" />

### 4. Configuración de DNS y FQDN
Se configuraron los servidores DNS del FortiGate (Network > DNS) apuntando a `8.8.8.8` y `8.8.4.4`. En *Policy & Objects > Addresses*, se creó un objeto tipo FQDN apuntando a `archive.ubuntu.com` para la DMZ, y un objeto tipo IP/Netmask para `8.8.8.8`.
<img width="1483" height="942" alt="imagen" src="https://github.com/user-attachments/assets/a05911cf-e619-46d0-967a-907c929faf7c" />


---

## Políticas de Seguridad (Firewall Policies)

Ubicación en GUI: *Policy & Objects > Firewall Policy*

| ID | Nombre | Incoming Interface | Outgoing Interface | Source | Destination | Service | Action | NAT |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **1** | `DMZ_TO_UPDATES` | `port2` (DMZ) | `port1` (WAN) | Subred DMZ (`10.20.24.0/28`) | FQDN: `archive.ubuntu.com`<br>IP: `8.8.8.8` | DNS, HTTP, HTTPS, PING | **ACCEPT** | **ENABLE** (Use Outgoing Interface IP) |
<img width="1485" height="1003" alt="imagen" src="https://github.com/user-attachments/assets/729e2bc3-e7db-4a15-8561-3876b28d0bb6" />

| **2** | `INTERNET_VLANS` | `VLAN10_LIMITADOS`<br>`VLAN20_SSH` | `port1` (WAN) | `all` | `all` | ALL | **ACCEPT** | **ENABLE** (Use Outgoing Interface IP) |
<img width="940" height="940" alt="imagen" src="https://github.com/user-attachments/assets/daf2d8af-f5fe-4b6d-88ba-1ef8fb423142" />

| **0** | `Implicit Deny` | `any` | `any` | `all` | `all` | ALL | **DENY** | DISABLE |
<img width="1485" height="54" alt="imagen" src="https://github.com/user-attachments/assets/cd1f8d55-ec86-4554-bc01-1a456414cbf0" />


---

## Validación y Pruebas Analíticas

Para certificar el correcto funcionamiento de toda la topología, se ejecutaron y grabaron en video las siguientes pruebas de validación:

### Fase 1: Validación de Borde (WAN)
**Objetivo:** Confirmar resolución DNS local y enrutamiento hacia el exterior.
**Comandos ejecutados (CLI FortiGate):**

execute ping 8.8.8.8
<img width="777" height="261" alt="imagen" src="https://github.com/user-attachments/assets/4bbf4af1-5a9d-49a5-a51f-691d94866d53" />

execute ping google.com
<img width="838" height="259" alt="imagen" src="https://github.com/user-attachments/assets/84d814ee-92e9-4216-bf3d-7e413f7f17ce" />

get router info routing-table all
<img width="889" height="351" alt="imagen" src="https://github.com/user-attachments/assets/193aefb7-4767-4ab1-bb6e-4e4950c96605" />

*Resultado Esperado:* 0% de pérdida de paquetes y presencia de la ruta estática predeterminada `S* 0.0.0.0/0`.

### Fase 2: Validación DMZ (Zero Trust y FQDN)
**Objetivo:** Confirmar que el servidor público solo alcanza su repositorio autorizado y no tiene navegación libre.
**Comandos ejecutados (Terminal Ubuntu DMZ):**

ping -c 4 8.8.8.8               # Prueba de DNS autorizado (Éxito)
<img width="784" height="286" alt="imagen" src="https://github.com/user-attachments/assets/1377b802-92c1-4548-8c2f-0e9660e5834b" />

ping -c 4 archive.ubuntu.com    # Prueba de FQDN autorizado (Éxito)
<img width="915" height="234" alt="imagen" src="https://github.com/user-attachments/assets/c7473fb9-d268-4dcc-a584-d6cd650b7df6" />

ping -c 4 google.com            # Prueba de tráfico no autorizado (Fallo / Drop)
<img width="625" height="34" alt="imagen" src="https://github.com/user-attachments/assets/d23bf6e2-6149-42a7-b0b9-59c8848d8d9f" />

*Resultado Esperado:* Resolución y respuesta exclusiva hacia los dominios permitidos en la regla `DMZ_TO_UPDATES`.

### Fase 3: Validación LAN (Múltiples Interfaces y SNAT)
**Objetivo:** Verificar interconexión VLAN-WAN y enmascaramiento de IPs.
**Comandos ejecutados (Terminal USER1 y USER2):**
```bash
ping -c 4 google.com
```
<img width="930" height="264" alt="imagen" src="https://github.com/user-attachments/assets/8edff97e-6f58-432a-bb26-73d05530e3f5" />
*Resultado Esperado:* Respuesta exitosa demostrando que la regla `INTERNET_VLANS` procesa ambas redes correctamente.

### Fase 4: Auditoría de Políticas (Hit Counts)
**Objetivo:** Demostrar gráficamente el procesamiento de paquetes.
**Acción:** Revisión de la columna "Bytes" en la tabla de *Firewall Policy*. 
<img width="1483" height="246" alt="imagen" src="https://github.com/user-attachments/assets/1b77f87d-0efc-48df-b84d-6e6f7764a498" />

*Resultado Esperado:* El contador de las políticas 1 y 2 debe registrar una cantidad de bytes mayor a cero (ej. `14.5 KB`), garantizando que las reglas configuradas están interceptando y permitiendo el tráfico real.

---

## Diagramas y Evidencias Visuales

**Topología Lógica Integrada en GNS3:**
<img width="966" height="493" alt="imagen" src="https://github.com/user-attachments/assets/2914494d-d36d-4390-86d6-a5d86d8dfff9" />


**Configuración de las Interfaces Lógicas (Subinterfaces/VLANs):**
<img width="1489" height="349" alt="imagen" src="https://github.com/user-attachments/assets/93120c27-7cc9-4267-bd30-0b437b3560be" />


**Tabla de Políticas de Firewall con Hit Counts (Bytes):**
<img width="1483" height="246" alt="imagen" src="https://github.com/user-attachments/assets/1b77f87d-0efc-48df-b84d-6e6f7764a498" />

**Pruebas de Conectividad Efectivas desde Ubuntu (DMZ / USER1):**
<img width="910" height="520" alt="imagen" src="https://github.com/user-attachments/assets/91ce7deb-3323-4e06-b087-39ad71894348" />

<img width="919" height="322" alt="imagen" src="https://github.com/user-attachments/assets/5a0149ea-27a0-45cf-9094-b4619879b48a" />

---

## Running Configurations Completas

### 1. Cisco Switch IOS (SWA)
Extracción de la configuración de memoria no volátil (NVRAM):
```text
SWA# show running-config
!
hostname SWA
!
vlan 10
 name LIMITADOS
!
vlan 20
 name SSH
!
interface GigabitEthernet0/0
 description ENLACE_TRONCAL_A_FORTIGATE
 switchport trunk encapsulation dot1q
 switchport mode trunk
!
interface GigabitEthernet0/1
 description ACCESO_VLAN10_USER1
 switchport access vlan 10
 switchport mode access
 spanning-tree portfast
!
interface GigabitEthernet0/2
 description ACCESO_VLAN20_USER2
 switchport access vlan 20
 switchport mode access
 spanning-tree portfast
!
interface GigabitEthernet0/3
 shutdown
!
end
```

### 2. FortiGate NGFW (Configuraciones Relevantes CLI)
Bloques de configuración extraídos del sistema operativo FortiOS:

**Subinterfaces VLAN:**
```text
config system interface
    edit "VLAN10_LIMITADOS"
        set vdom "root"
        set ip 10.10.10.1 255.255.255.0
        set allowaccess ping
        set interface "port3"
        set vlanid 10
    next
    edit "VLAN20_SSH"
        set vdom "root"
        set ip 10.20.20.1 255.255.255.0
        set allowaccess ping ssh
        set interface "port3"
        set vlanid 20
    next
    edit "port2"
        set vdom "root"
        set ip 10.20.24.1 255.255.255.240
        set allowaccess ping
    next
end
```

**Configuración Global (Múltiples Interfaces):**
```text
config system settings
    set gui-multiple-interface-policy enable
end
```

---

## Scripts y Archivos Utilizados

### Configuración de Red Netplan (Servidor Ubuntu DMZ)
Se utilizó el administrador de red moderno de Linux (`netplan`) para fijar el enrutamiento estático del servidor DMZ hacia el gateway del FortiGate.

**Archivo modificado:** `/etc/netplan/00-installer-config.yaml`
```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    eth0:
      addresses:
        - 10.20.24.2/28
      routes:
        - to: default
          via: 10.20.24.1
      nameservers:
        addresses:
          - 8.8.8.8
          - 8.8.4.4
```
**Comando de ejecución:** `sudo netplan apply`

### Configuración de Red Netplan (USER1 - VLAN 10)
```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    eth0:
      addresses:
        - 10.10.10.2/24
      routes:
        - to: default
          via: 10.10.10.1
      nameservers:
        addresses:
          - 8.8.8.8
          - 8.8.4.4
```

### Configuración de Red Netplan (USER2 - VLAN 20)
```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    eth0:
      addresses:
        - 10.20.20.2/24
      routes:
        - to: default
          via: 10.20.20.1
      nameservers:
        addresses:
          - 8.8.8.8
          - 8.8.4.4
```

---

## Conclusión

La arquitectura desplegada en GNS3 cumple exhaustivamente con las normativas de seguridad empresarial y diseño de redes. Al delegar el enrutamiento inter-VLAN al FortiGate (Router-on-a-Stick), se centraliza la inspección de tráfico y se garantiza un control granular de Capa 3 a Capa 7.

El uso crítico de la interfaz `eth1` para el nodo Cloud resolvió de manera definitiva los históricos problemas de enrutamiento y NAT de las herramientas de virtualización de escritorio (VMware/VirtualBox) dentro de GNS3. Asimismo, la implementación estricta de objetos FQDN en las reglas del firewall para la zona DMZ demostró la efectividad del filtrado de egreso (Egress Filtering), protegiendo la red de posibles pivotes de ataque, fugas de datos (Data Exfiltration) y garantizando el principio de menor privilegio.
