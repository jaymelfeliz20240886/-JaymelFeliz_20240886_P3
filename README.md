# Laboratorio de Seguridad Perimetral: FortiGate, DMZ y Segmentación de VLANs en GNS3

🎥 **[Ver Video Demostrativo del Laboratorio Aquí](LINK_A_TU_VIDEO_AQUI)**

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

## Tabla de Direccionamiento IP

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
