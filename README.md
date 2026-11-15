# SecureCorp-Lab

Laboratorio de infraestructura híbrida Microsoft con enfoque en seguridad, construido por niveles en Hyper-V.

## Escenario

SecureCorp es una empresa ficticia de 40 personas en Lima, con áreas de Gerencia, Contabilidad, Operaciones y TI. Construyo su infraestructura desde cero, nivel por nivel: identidades centralizadas, red segmentada, monitoreo y extensión a Azure. 

Cada nivel parte de una necesidad del negocio y se diseña con mínimo privilegio, segmentación y recuperación probada.

## Niveles

| Nivel | Tema | Estado |
|---|---|---|
| [1 · La oficina](nivel-1/) | Active Directory, DNS, DHCP, GPO, permisos NTFS, backup | 🔄 En curso |
| 2 · La red segura | pfSense por zonas, VPN, Linux, Wireshark | ⏳ Pendiente |
| 3 · Operación y vigilancia | Zabbix, eventos de seguridad, PowerShell, CI/CD, AZ-900 | ⏳ Pendiente |

## Entorno

| Componente | Detalle |
|---|---|
| Hipervisor | Hyper-V sobre Windows 11 Pro |
| Host | Intel Core i5-11400 · 32 GB RAM · NVMe para VMs |
| Servidores | Windows Server 2025 (evaluación) · Ubuntu Server 26.04 (Nivel 2) |
| Clientes | Windows 11 Enterprise (evaluación) |
| Red | `10.0.0.0/16` dividida en zonas `/24` · [ver plan IP](nivel-1/plan-ip.md)  |

## Autor

**Alexander Crisóstomo** · 4 años administrando infraestructura en el sector industrial-minero
(Active Directory, permisos NTFS, respaldos y red de dos sedes).
Me oriento a infraestructura Microsoft híbrida con enfoque en seguridad, sobre todo
en identidades y accesos. Preparando AZ-900.

[LinkedIn](https://www.linkedin.com/in/alexander-crisostomo) · [GitHub](https://github.com/alexcrisdev)
