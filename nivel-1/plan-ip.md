# Plan IP · SecureCorp-Lab · Nivel 1

> Fecha: 14/11/2026
> Comando para ver la red de mi casa: `ipconfig` (revisar el adaptador de red real, no los de Hyper-V)

Red de mi casa (para no repetirla): 192.168.1.0/24

## Nivel 1 · Red de la oficina

| Elemento                           | Valor |
|------------------------------------|-------|
| Subred del Nivel 1                 |10.0.10.0/24       |
| Máscara (decimal y CIDR)           |255.255.255.0 (/24)       |
| Gateway (futuro pfSense)           |10.0.10.1       |
| DC01 (IP fija)                     |10.0.10.10       |
| DNS preferido de DC01              |10.0.10.10       |
| DNS alternativo de DC01            |127.0.0.1       |
| Rango de IPs fijas para servidores |10.0.10.10 – 10.0.10.49 (.10–.19 identidad · .20–.29 monitoreo · .30-.49 otros) |
| Rango DHCP para clientes           |10.0.10.100-10.0.10.199 (temporal, solo en el nivel 1) |
| IPs reservadas / libres            |10.0.10.2 – .9 reservadas para red · 10.0.10.50 – .99 y .200 - .254 libres |

## Equipos del laboratorio

| Equipo    | Rol                       | IP       | Tipo (fija / DHCP) |
|-----------|---------------------------|----------|--------------------|
| DC01      | AD DS, DNS, DHCP          | 10.0.10.10         | fija     |
| CLIENTE01 | Equipo de usuario         | 10.0.10.100–.199   | DHCP     |
| pfSense   | Firewall / gateway (Nivel 2) | 10.0.10.1| fija |
| UBUNTU01  | Linux + Zabbix (Nivel 2-3)| 10.0.10.20         | fija     |

## Zonas previstas para el Nivel 2

| Zona       | Subred    | Para qué sirve |
|------------|-----------|----------------|
| SERVIDORES | 10.0.10.0/24 | Servirá para los servidores para gestionar identidad, monitoreo entre otros               |
| USUARIOS   | 10.0.20.0/24 | Servirá para organizar a los usuarios como sus equipos corporativos, impresoras, etc              |
| ADMIN      | 10.0.30.0/24 | Los equipos desde donde TI administra los servidores (PAW) y separar de la red de usuarios para que no se extienda una posible exposición frente a un hackeo               |
| INVITADOS  | 10.0.40.0/24 | Servirá para que solo tenga salida a internet, sin acceso a ninguna zona interna               |

## Justificación (2–3 líneas)

¿Por qué elegí este rango y esta organización?

Azure propone 10.0.0.0/16 por defecto; la nube usará 10.100.0.0/16 para no solaparse, además el tercer octeto identificaría la zona, deja huecos para crecer y coindirá con el ID de la VLAN en el nivel 2.

## Pregunta trampa: ¿qué DNS usa DC01 y por qué?

Usaría la 10.0.10.10 porque él es el DNS que conoce securecorp.local. Si se coloca la 8.8.8.8, no encontraría los registros SRV, no se podrá unir al dominio y no aplica GPO.
