# Semana 1 · Teoría: virtualización y diseño

> SecureCorp-Lab · Nivel 1 · La oficina
> Fecha: 14/11/2026

Cada concepto tiene tres niveles:
- 🟢 **En simple:** la idea con una analogía cotidiana.
- 🔵 **En técnico:** cómo se dice y funciona de verdad.
- 🎤 **En entrevista:** la respuesta lista para decir en voz alta.

> **Estructura para responder en entrevistas:** concepto → término técnico → consecuencia para el negocio.

---

## 1. Virtualización e hipervisor

🟢 **En simple:** un hipervisor es como el **administrador de un edificio**. El edificio (el PC físico) se divide en departamentos (las VMs). Cada inquilino cree que tiene su propia casa, con su luz y su agua, pero el administrador es quien reparte los recursos y evita que un vecino entre a la casa de otro.

🔵 **En técnico:** la **virtualización** permite ejecutar varios sistemas operativos aislados sobre un mismo equipo físico. El **hipervisor** es el software que crea las máquinas virtuales, les reparte CPU, RAM, disco y red, y las mantiene aisladas entre sí.

### Tipo 1 vs. tipo 2

| | 🟢 En simple | 🔵 En técnico | Ejemplos |
|---|---|---|---|
| **Tipo 1 (bare-metal)** | El administrador vive en los **cimientos** del edificio | Corre directamente sobre el hardware | Hyper-V, VMware ESXi, Proxmox |
| **Tipo 2 (alojado)** | El administrador es **un inquilino más** que subarrienda su departamento | Corre como un programa dentro de un sistema operativo | VirtualBox, VMware Workstation |

### Pregunta 1 · "Hyper-V es tipo 2 porque se instala dentro de Windows"

🟢 **En simple:** parece que lo instalas "dentro" de Windows, pero al activarlo, el administrador baja a los cimientos y tu Windows pasa a ser **el departamento del conserje**: el más importante, pero un departamento más.

🔵 **En técnico:** al activar Hyper-V, el hipervisor arranca **primero**, directamente sobre el hardware. El Windows del host pasa a correr encima como **partición raíz** (*root partition*): una VM privilegiada que administra a las demás y controla los drivers. Cada VM que creas es una **partición hija**.

🎤 **En entrevista:** "No, Hyper-V es tipo 1. Al activarlo, el hipervisor arranca directamente sobre el hardware y Windows pasa a correr encima como partición raíz. Que se active desde Windows no lo convierte en tipo 2. Por eso es la misma tecnología que usa Azure."

### Glosario de virtualización

| Término | 🟢 En simple | 🔵 En técnico |
|---|---|---|
| **Host (anfitrión)** | El edificio | Equipo físico que ejecuta el hipervisor. En el laboratorio, mi PC. |
| **Guest (invitado)** | Un inquilino | Sistema operativo que corre dentro de una VM. Ejemplo: DC01. |
| **Partición raíz** | El departamento del conserje | El Windows del host en Hyper-V: VM privilegiada que administra las demás. |
| **Partición hija** | Cada departamento alquilado | Cada VM creada en Hyper-V. |
| **VT-x / AMD-V** | Los cimientos preparados para construir departamentos | Extensiones del procesador para virtualizar por hardware. Se activan en la BIOS/UEFI. |
| **VHDX** | El "disco" de cada departamento, que en realidad es un archivo | Formato de disco virtual de Hyper-V: hasta 64 TB, más resistente a la corrupción que el VHD. |
| **Disco dinámico** | Una mochila que se estira según lo que metes | Crece según el uso. Ahorra espacio. |
| **Disco fijo** | Una maleta del tamaño máximo desde el inicio | Reserva todo el tamaño de entrada. Rinde un poco más. |
| **Memoria dinámica** | El administrador presta más RAM cuando el inquilino la necesita | Asigna RAM entre un mínimo y un máximo según la demanda. **Se desactiva en los DCs** para tener RAM estable. |
| **Integration Services** | El intercomunicador entre el conserje y los departamentos | Componentes para apagado limpio, sincronización de hora, VSS, etc. |

---

## 2. Switches virtuales

🟢 **En simple:** el switch virtual son los **pasillos del edificio**. Según cómo los construyas, los inquilinos pueden salir a la calle, visitar solo al conserje o hablar solo entre ellos.

🔵 **En técnico:** un switch virtual es un switch de red implementado por software. Las tarjetas de red virtuales (**vNIC**) de las VMs se conectan a él.

| Tipo | 🟢 En simple | 🔵 En técnico | Uso típico |
|---|---|---|---|
| **Externo** | Pasillo con salida a la **calle** | VMs + host + red física (router, internet) | Cuando la VM debe ser un equipo más de la red real. Ejemplo: la WAN de pfSense. |
| **Interno** | Pasillo que conecta los departamentos **y la oficina del conserje**, sin salida a la calle | VMs + host, sin red física | Laboratorios aislados administrados desde el host |
| **Privado** | Pasillo **solo entre departamentos**: ni el conserje entra | Solo VMs entre sí | Segmentos aislados detrás de un firewall (Nivel 2) |

Al crear un switch **interno**, aparece en el host una tarjeta nueva: **`vEthernet (nombre-del-switch)`**. Es la "puerta" del conserje a ese pasillo.

### Pregunta 3 · VMs que se vean entre sí, pero no con mi PC

🟢 **En simple:** un pasillo solo entre departamentos, sin puerta hacia el conserje.

🔵 **En técnico:** un switch **privado**: comunica solo a las VMs y deja fuera al host y a la red física.

🎤 **En entrevista:** "Un switch privado. Es el más aislado: solo comunica VMs entre sí. Se usa para segmentos que solo deben alcanzarse a través de un firewall virtual."

---

## 3. DHCP, rogue DHCP y APIPA

### DHCP

🟢 **En simple:** el DHCP es la **recepción de un hotel**: llegas y te asignan una habitación (IP), te dicen dónde está la salida (gateway) y a quién preguntar por direcciones (DNS).

🔵 **En técnico:** protocolo que asigna automáticamente IP, máscara, gateway y DNS a los equipos de la red.

### Pregunta 2 · DC01 con DHCP conectado a un switch externo

🟢 **En simple:** instalas una **recepción falsa** en la entrada de tu casa. Tus familiares llegan, ella les da una habitación y les indica una salida que no existe. Se quedan sin internet.

🔵 **En técnico:** DC01 sería un **rogue DHCP** (DHCP no autorizado) en la red real. Los equipos podrían recibir su configuración de DC01 en lugar del router, con un **gateway y un DNS incorrectos**, y perder conectividad. Un atacante usa esta misma técnica para hacer **man-in-the-middle**: se pone como gateway o DNS y ve todo el tráfico.
Se mitiga con **DHCP Snooping** en los switches físicos y con la **autorización de DHCP en Active Directory**.

🎤 **En entrevista:** "Sería un rogue DHCP. Los equipos de la red real podrían recibir un gateway y un DNS equivocados y quedarse sin internet; en una empresa, sería un incidente que deja a los usuarios sin red. También es una técnica de man-in-the-middle. Se previene con DHCP Snooping y autorizando los DHCP en AD. Por eso el laboratorio usa un switch interno."

### APIPA

🟢 **En simple:** llegas al hotel y **no hay nadie en recepción**. Te asignas una habitación provisional por tu cuenta. Puedes hablar con otros huéspedes que hicieron lo mismo, pero no sabes dónde está la salida.

🔵 **En técnico:** **APIPA** (*Automatic Private IP Addressing*): cuando un equipo configurado con DHCP no recibe respuesta de ningún servidor DHCP, Windows se asigna una IP del rango **`169.254.0.0/16`** (*link-local*). Solo sirve para comunicarse con equipos del mismo segmento que también tengan APIPA. **No tiene gateway ni sale a internet.**

🎤 **En entrevista:** "Si un usuario no tiene red y en ipconfig veo una 169.254.x.x, es APIPA: el equipo no recibió respuesta del DHCP. Entonces reviso el servidor DHCP, el cable, la VLAN o si el ámbito se quedó sin direcciones. No es un problema de DNS ni del navegador."

> **Visto en el laboratorio:** al crear `SecureCorp-LAN`, el host recibió `169.254.214.118`, porque en ese switch todavía no existe un DHCP.

### Glosario de red

| Término | 🟢 En simple | 🔵 En técnico |
|---|---|---|
| **Gateway** | La puerta de salida del barrio | Equipo que saca el tráfico fuera de la subred local (router o firewall) |
| **IP estática** | Una dirección fija, que no cambia nunca | IP configurada a mano. La usan servidores, DCs y firewalls. |
| **vNIC** | La puerta de cada departamento hacia el pasillo | Tarjeta de red virtual de una VM |
| **DHCP Snooping** | Un guardia que solo deja trabajar a la recepción oficial | Función del switch que bloquea respuestas DHCP desde puertos no autorizados |

---

## 4. Firmware y arranque seguro

🟢 **En simple:** el firmware es el **guardia de la puerta** que revisa quién entra cuando el equipo arranca. La BIOS es un guardia antiguo que deja pasar a cualquiera. UEFI con Secure Boot es un guardia moderno que **pide credencial firmada**.

| Término | 🟢 En simple | 🔵 En técnico |
|---|---|---|
| **BIOS** | Guardia antiguo, sin control de identidad | Firmware de arranque antiguo: discos MBR de hasta 2 TB y sin Secure Boot |
| **UEFI** | Guardia moderno | Firmware que reemplaza a la BIOS: discos GPT, arranque rápido y Secure Boot |
| **Secure Boot** | Solo entra quien tiene credencial firmada | Solo arranca cargadores y sistemas **firmados digitalmente**. Bloquea *bootkits* y *rootkits*. |
| **TPM** | Una caja fuerte pequeña soldada al equipo | Chip (físico o virtual) que guarda claves criptográficas protegidas. Permite **BitLocker**, el **arranque medido** y Windows Hello. Windows 11 exige TPM 2.0. |
| **vTPM** | La misma caja fuerte, pero virtual | TPM virtual que Hyper-V ofrece a las VMs de Generación 2. Su estado se guarda en el archivo `.vmgs`. |
| **Trusted Launch** | Lo mismo, en la nube | VMs de Azure con Secure Boot y vTPM |

### Generación 1 vs. Generación 2

| | 🟢 En simple | 🔵 En técnico |
|---|---|---|
| **Generación 1** | Departamento antiguo con guardia viejo | BIOS emulada, disco IDE, sin Secure Boot ni TPM. Para sistemas antiguos. |
| **Generación 2** | Departamento moderno con guardia y caja fuerte | UEFI, Secure Boot, vTPM y disco SCSI virtual. La opción para sistemas modernos. |

⚠️ **La generación no se puede cambiar** después de crear la VM.

### Pregunta 4 · CLIENTE01 creado como Gen 1 y Windows 11 no se instala

🟢 **En simple:** Windows 11 solo acepta mudarse a departamentos con guardia moderno y caja fuerte. El departamento Gen 1 no los tiene, y no se pueden instalar después: hay que alquilar otro.

🔵 **En técnico:** Windows 11 exige **UEFI con Secure Boot y TPM 2.0**. La Generación 1 usa BIOS y no ofrece vTPM. Como la generación no se puede cambiar, hay que crear la VM de nuevo como **Generación 2** y activar el TPM virtual.

🎤 **En entrevista:** "Windows 11 exige UEFI, Secure Boot y TPM 2.0, y la Generación 1 no los ofrece. La generación no se puede cambiar después, así que hay que recrear la VM como Generación 2 con vTPM."

---

## 5. Checkpoints y backups

### Checkpoint

🟢 **En simple:** un checkpoint es como **guardar la partida** en un videojuego antes de enfrentar al jefe final. Si pierdes, vuelves a ese punto. Pero la partida guardada está **en la misma consola**: si se quema la consola, pierdes el juego y las partidas.

🔵 **En técnico:** "foto" del estado de una VM en un momento dado. Al crearlo, el disco original (`.vhdx`) queda congelado y los cambios se escriben en un disco diferencial (`.avhdx`). Volver atrás es descartar ese diferencial.

| Tipo | 🟢 En simple | 🔵 En técnico |
|---|---|---|
| **Estándar** | Guarda la partida con el juego en pausa, justo en ese segundo | Guarda el disco **y la RAM**. Restaura al instante exacto, con los programas abiertos. |
| **Producción** | Guarda la partida después de salir al menú, de forma ordenada | Usa **VSS** dentro del invitado: estado consistente, como un apagado limpio. **Recomendado para servidores.** |

### Pregunta 5 · "Tenemos checkpoints, no necesitamos backups"

🟢 **En simple:** las partidas guardadas viven en la misma consola. Si se quema, te roban o un virus la borra, lo pierdes todo. Además, cada partida guardada depende de la partida original: sin ella no sirve.

🔵 **En técnico:**
- **Están en el mismo almacenamiento que la VM.** Si falla el disco del host, alguien borra la VM o un ransomware cifra el servidor, se pierden la VM y sus checkpoints a la vez. No cumple la regla **3-2-1**.
- **Dependen del disco original.** El `.avhdx` solo guarda los cambios: sin el `.vhdx` padre no sirve. Además, acumularlos baja el rendimiento, y en un DC restaurar uno antiguo puede causar un **USN rollback**.

🎤 **En entrevista:** "Un checkpoint no es un backup. Vive en el mismo almacenamiento que la VM, así que un fallo de disco o un ransomware se lleva ambos, y no cumple 3-2-1. Además depende del disco padre y degrada el rendimiento. En un DC, restaurar un checkpoint antiguo puede causar un USN rollback. Los checkpoints son para volver atrás antes de un cambio, no para recuperación."

### Glosario de respaldo

| Término | 🟢 En simple | 🔵 En técnico |
|---|---|---|
| **VSS** | Pedir a todos que dejen de escribir un segundo para sacar la foto | *Volume Shadow Copy Service*: congela las escrituras para copiar datos consistentes, aunque estén en uso |
| **Backup** | Una copia de tus partidas guardada **en otra casa** | Copia independiente, en otro medio o lugar, recuperable aunque se pierda el sistema original |
| **Regla 3-2-1** | 3 copias, en 2 lugares de distinto tipo, y 1 lejos de casa | **3** copias de los datos, en **2** medios distintos, **1** fuera del sitio. La versión **3-2-1-1-0** suma 1 copia inmutable o desconectada (contra ransomware) y 0 errores al probar la restauración. |
| **RPO** | ¿Cuánto trabajo puedo permitirme perder? | *Recovery Point Objective*: pérdida máxima de datos aceptable, en tiempo. RPO 24 h = backup diario. |
| **RTO** | ¿Cuánto tiempo puedo estar sin funcionar? | *Recovery Time Objective*: tiempo máximo para restaurar el servicio |
| **USN rollback** | Volver un registro de asistencia a una versión vieja: los demás ya no confían en él | Al restaurar un DC desde una copia antigua en una red con varios DCs, los números de secuencia (USN) se desincronizan y la replicación se rompe |
| **VM-Generation ID** | Un sello que cambia cuando "viajas en el tiempo" | Identificador que Hyper-V cambia al restaurar un checkpoint. Windows Server lo usa para proteger AD del USN rollback. |

---

## 6. Aprendido en la práctica

### Sesión mejorada de Hyper-V
🟢 **En simple:** en vez de mirar la VM por una ventanita, entras por un **pasillo privado** directo, con más comodidades (copiar y pegar, mejor resolución).

🔵 **En técnico:** la **sesión mejorada** (*Enhanced Session Mode*) usa **RDP** sobre el **VMBus**, un canal interno entre el host y la VM que **no pasa por la red**. Por eso funciona aunque la VM no tenga IP. En sesión mejorada, Ctrl+Alt+Supr se envía con **Ctrl+Alt+Fin**.

🔐 El portapapeles compartido es un **canal de fuga de datos**. En empresas suele bloquearse en servidores críticos.

### Ping: "unreachable" vs. "timeout"
🟢 **En simple:**
- **Unreachable:** tocas el timbre y **nadie vive ahí**.
- **Timeout:** alguien vive ahí, pero **no abre la puerta** (un firewall lo impide).

🔵 **En técnico:**

| Mensaje | Qué pasó | Qué indica |
|---|---|---|
| `Reply from <mi IP>: Destination host unreachable` | La consulta **ARP** ("¿quién tiene esta IP?") no tuvo respuesta. El ping ni siquiera salió. | No existe un equipo con esa IP en el segmento local |
| `Request timed out` | El paquete salió, pero no volvió respuesta | El equipo existe pero un firewall bloquea ICMP, o la respuesta se perdió |

🎤 **En entrevista:** "Si el unreachable lo responde mi propia IP, falló ARP: el destino no está en el segmento. Si es timeout, lo más probable es un firewall bloqueando ICMP."

> **Visto en el laboratorio:** `ping 10.0.10.1` desde DC01 dio *unreachable* porque pfSense todavía no existe.

### Datos útiles para reconocer en el trabajo

| Dato | 🟢 En simple | 🔵 En técnico |
|---|---|---|
| MAC que empieza con **`00-15-5D`** | La "marca de fábrica" de las VMs de Hyper-V | Identificador de fabricante (OUI) de Microsoft Hyper-V. Si lo ves en la red, es una VM de Hyper-V. |
| Números al inicio de un ISO | La versión real de Windows, escrita en código | Número de **compilación**: 17763 = Server 2019 · 20348 = Server 2022 · 26100 = Server 2025 |
| Evento **Security-SPP 1014** | Ruido: el servicio de licencias se queja de que no tiene internet | Error de activación esperado en un laboratorio aislado. Saber distinguir el ruido de los errores reales es clave para el Nivel 3. |
| Archivo **`.vmgs`** | La caja donde la VM guarda sus secretos | *VM Guest State*: guarda el estado del vTPM y del firmware UEFI de la VM |

---

## 7. Decisiones para SecureCorp-Lab

| Decisión | Elección | Motivo |
|---|---|---|
| Hipervisor | Hyper-V (incluido en Windows 11 Pro) | Es tipo 1, gratuito y del mismo ecosistema que Windows Server y Azure |
| Switch del Nivel 1 | Interno (`SecureCorp-LAN`) | Aísla el DHCP del laboratorio de la red real y permite administrar las VMs desde el host |
| Switch `WAN` (existente) | Externo, se conserva | Será la WAN de pfSense en el Nivel 2. **Nunca** se conectan ahí DC01 ni CLIENTE01. |
| Switches del Nivel 2 | Privados por zona (previsto) | pfSense será el único punto de unión entre zonas |
| Generación de las VMs | Generación 2 con vTPM | Secure Boot y vTPM. CLIENTE01 (Windows 11) lo exige. |
| RAM de DC01 | 4 GB fijos, sin memoria dinámica | Recomendado para controladores de dominio |
| Ubicación de las VMs | `C:\Lab\VMs` (NVMe) | Mejor rendimiento de disco |
| ISOs y backups | Disco E: (HDD) | Ahorra espacio en el NVMe y separa los backups del disco de las VMs |
| Checkpoints | Solo antes de cambios grandes; se borran al validar | No reemplazan al backup de la semana 4 |
