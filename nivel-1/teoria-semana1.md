# Semana 1 · Teoría: virtualización y diseño

> SecureCorp-Lab · Nivel 1 · La oficina
> Fecha: 14/11/2026

---

## 1. Preguntas de repaso (versión de entrevista)

**1. Un compañero dice que Hyper-V es tipo 2 "porque se instala dentro de Windows". ¿Qué le respondes?**

No, Hyper-V es un hipervisor **tipo 1 (bare-metal)**. Al activarlo, el hipervisor arranca directamente sobre el hardware y Windows pasa a correr encima como **partición raíz**, una VM privilegiada que administra a las demás. Que se active desde Windows no lo convierte en tipo 2.

**2. ¿Qué podría pasar si conectas DC01, con DHCP activo, a un switch externo?**

DC01 se convertiría en un **rogue DHCP** (DHCP no autorizado) dentro de la red real. Los equipos podrían recibir su IP de DC01 en lugar del router, junto con un **gateway y un DNS incorrectos**, y quedarse sin internet. En una empresa sería un incidente que deja a los usuarios sin red. Por eso el laboratorio usa un switch **interno**.

**3. Quieres que las VMs se vean entre sí, pero que ni tu PC pueda conectarse a ellas. ¿Qué switch usas?**

Un switch **privado**: solo comunica a las VMs entre sí y deja fuera al host y a la red física.

**4. Creaste CLIENTE01 como Generación 1 y Windows 11 no se instala. ¿Por qué pasa y cómo lo arreglas?**

Windows 11 exige **UEFI con Secure Boot y TPM 2.0**. La Generación 1 usa BIOS y no ofrece TPM virtual. Como la generación **no se puede cambiar** después de crear la VM, toca crearla de nuevo como **Generación 2** y activar el TPM virtual.

**5. Tu jefe dice: "tenemos checkpoints de todos los servidores, no necesitamos backups". Dale dos razones por las que está equivocado.**

- **Están en el mismo almacenamiento que la VM.** Si falla el disco del host, alguien borra la VM o un ransomware cifra el servidor, se pierden la VM y sus checkpoints a la vez. No cumple la regla **3-2-1**.
- **Dependen del disco original.** Un checkpoint solo guarda los cambios (`.avhdx`), así que sin el disco padre (`.vhdx`) no sirve. Además, acumularlos baja el rendimiento, y en un controlador de dominio restaurar uno antiguo puede causar un **USN rollback**.

> **Estructura para responder en entrevistas:** concepto → término técnico → consecuencia para el negocio.

---

## 2. Glosario

### Virtualización

| Término | Definición |
|---|---|
| **Virtualización** | Ejecutar varios sistemas operativos aislados sobre un mismo equipo físico, repartiendo CPU, RAM, disco y red. |
| **Hipervisor** | Software que crea y ejecuta las máquinas virtuales y las aísla entre sí. |
| **Hipervisor tipo 1 (bare-metal)** | Corre directamente sobre el hardware. Ejemplos: Hyper-V, VMware ESXi, Proxmox. Es lo que se usa en empresas y en la nube. |
| **Hipervisor tipo 2 (alojado)** | Corre como un programa dentro de un sistema operativo. Ejemplos: VirtualBox, VMware Workstation. |
| **Host (anfitrión)** | El equipo físico que ejecuta el hipervisor. En el laboratorio, mi PC. |
| **Guest (invitado)** | Sistema operativo que corre dentro de una VM. Ejemplo: DC01. |
| **Partición raíz (root partition)** | En Hyper-V, la instancia de Windows del host. Corre sobre el hipervisor como una VM privilegiada: administra las demás VMs y controla los drivers del hardware. |
| **Partición hija (child partition)** | Cada VM que creo en Hyper-V. |
| **VT-x / AMD-V** | Extensiones del procesador para virtualización por hardware. Se activan en la BIOS/UEFI y son obligatorias para Hyper-V. |
| **VHDX** | Formato de disco virtual de Hyper-V. Admite hasta 64 TB y resiste mejor la corrupción que el antiguo VHD. |
| **Disco dinámico vs. fijo** | El dinámico crece según se usa (ahorra espacio). El fijo reserva todo el tamaño desde el inicio (rinde un poco más). |
| **Memoria dinámica** | Hyper-V asigna RAM a la VM según la necesite, entre un mínimo y un máximo. |
| **Integration Services** | Componentes que mejoran la comunicación entre el host y la VM: apagado limpio, sincronización de hora, copias con VSS, etc. |

### Redes virtuales

| Término | Definición |
|---|---|
| **Switch virtual** | Switch de red implementado por software. Las tarjetas de red virtuales de las VMs se conectan a él. |
| **Switch externo** | Conecta las VMs, el host y la red física (router, internet). Las VMs son un equipo más de la red real. |
| **Switch interno** | Conecta las VMs y el host, sin acceso a la red física. Ideal para laboratorios aislados. |
| **Switch privado** | Conecta solo a las VMs entre sí. Ni el host las ve. Se usa para segmentos aislados detrás de un firewall. |
| **NIC virtual (vNIC)** | Tarjeta de red de una VM. |
| **DHCP** | Protocolo que asigna automáticamente IP, máscara, gateway y DNS a los equipos de la red. |
| **Rogue DHCP** | Servidor DHCP no autorizado en una red. Reparte configuraciones incorrectas y deja equipos sin conectividad. También se puede usar para ataques de intermediario (*man-in-the-middle*), porque el atacante se pone como gateway o DNS. Se mitiga con **DHCP Snooping** en los switches y, en dominios, con la **autorización de DHCP en Active Directory**. |
| **Gateway (puerta de enlace)** | Equipo que saca el tráfico fuera de la subred local, normalmente un router o firewall. |
| **IP estática** | IP configurada a mano. Los servidores (DC, DNS, firewall) siempre la usan para que su dirección no cambie. |

### Firmware y arranque seguro

| Término | Definición |
|---|---|
| **BIOS** | Firmware antiguo de arranque. Limitado: discos MBR de hasta 2 TB y sin Secure Boot. |
| **UEFI** | Firmware moderno que reemplaza a la BIOS. Admite discos GPT, arranque más rápido y Secure Boot. |
| **Secure Boot** | Función de UEFI que solo permite arrancar cargadores y sistemas firmados digitalmente por fabricantes de confianza. Bloquea *bootkits* y *rootkits*. |
| **TPM (Trusted Platform Module)** | Chip de seguridad (físico o virtual) que guarda claves criptográficas de forma protegida. Permite el cifrado de disco con **BitLocker**, el **arranque medido** (verifica que el arranque no fue alterado) y Windows Hello. Windows 11 exige TPM 2.0. |
| **vTPM** | TPM virtual que Hyper-V ofrece a las VMs de Generación 2. |
| **Generación 1** | VM con BIOS emulada y disco IDE. Para sistemas antiguos. Sin Secure Boot ni TPM. |
| **Generación 2** | VM con UEFI, Secure Boot, vTPM y disco SCSI virtual. Es la opción por defecto para sistemas modernos. No se puede cambiar después de crear la VM. |
| **Trusted Launch** | Equivalente en Azure: VMs con Secure Boot y vTPM. |

### Checkpoints y respaldo

| Término | Definición |
|---|---|
| **Checkpoint** | "Foto" del estado de una VM en un momento dado. Sirve para volver atrás rápido antes de un cambio riesgoso. **No es un backup.** |
| **AVHDX** | Disco diferencial que crea el checkpoint. Guarda solo los cambios hechos después de la foto y depende del VHDX padre. |
| **Checkpoint estándar** | Guarda el disco y la memoria RAM. Restaura la VM al instante exacto, con los programas abiertos. |
| **Checkpoint de producción** | Usa VSS dentro del invitado y guarda un estado consistente, como si la VM se hubiera apagado limpiamente. Es el recomendado para servidores. |
| **VSS (Volume Shadow Copy Service)** | Servicio de Windows que congela momentáneamente las escrituras para copiar datos de forma consistente, aunque los archivos estén en uso. |
| **Backup** | Copia independiente de los datos, guardada en otro medio o lugar, que permite recuperarlos aunque se pierda el sistema original. |
| **Regla 3-2-1** | **3** copias de los datos (la original y 2 respaldos), en **2** tipos de medio distintos (por ejemplo disco y nube), y **1** copia fuera del sitio. Su versión moderna, **3-2-1-1-0**, agrega 1 copia inmutable o desconectada (contra ransomware) y 0 errores en las pruebas de restauración. |
| **RPO (Recovery Point Objective)** | Cuántos datos puede perder la empresa como máximo, medido en tiempo. Ejemplo: un RPO de 24 h exige un backup diario. |
| **RTO (Recovery Time Objective)** | En cuánto tiempo se debe restaurar el servicio después de una caída. |
| **USN rollback** | Problema de Active Directory que ocurre al restaurar un controlador de dominio desde una copia antigua (checkpoint o clon) en una red con varios DCs. Los números de secuencia (USN) quedan desincronizados, la replicación se rompe y aparecen inconsistencias. |
| **VM-Generation ID** | Identificador que Hyper-V cambia cuando una VM se restaura desde un checkpoint. Windows Server lo usa para detectar la restauración y proteger AD del USN rollback. |

---

## 3. Decisiones para SecureCorp-Lab

| Decisión | Elección | Motivo |
|---|---|---|
| Hipervisor | Hyper-V (incluido en Windows 11 Pro) | Es tipo 1, gratuito y el mismo ecosistema que Windows Server y Azure. |
| Switch del Nivel 1 | Interno (`SecureCorp-LAN`) | Aísla el DHCP del laboratorio de la red real y permite administrar las VMs desde el host. |
| Switches del Nivel 2 | Privados por zona (previsto) | pfSense será el único punto de unión entre zonas. |
| Generación de las VMs | Generación 2 | Secure Boot y vTPM. CLIENTE01 (Windows 11) lo exige. |
| Ubicación de las VMs | `C:\Lab\VMs` (NVMe) | Mejor rendimiento de disco. |
| ISOs y backups | Disco E: (HDD) | Ahorra espacio en el NVMe y separa los backups del disco de las VMs. |
| Checkpoints | Solo antes de cambios grandes; se borran al validar | No reemplazan al backup de la semana 4. |
APIPA -> cuando un equipo configurado con DHCP no recibe respuesta de ningún servidor DHCP, Windows se asigna solo una IP del rango 169.254.0.0/16 (link-local). Solo sirve para comunicarse con otros equipos del mismo segmento que también tengan APIPA. No tiene gateway ni sale a internet.