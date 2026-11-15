# Semana 2 · Teoría: Active Directory, DNS y Kerberos

> SecureCorp-Lab · Nivel 1 · La oficina
> Fecha: 15/11/2026

Cada concepto tiene tres niveles:
- 🟢 **En simple:** la idea con una analogía cotidiana.
- 🔵 **En técnico:** cómo se dice y funciona de verdad.
- 🎤 **En entrevista:** la respuesta lista para decir en voz alta.

---

## 1. Active Directory (AD DS)

🟢 **En simple:** es la **lista oficial de personal** de la empresa, guardada en un solo lugar. Cada empleado tiene una sola credencial que le abre todas las puertas que le corresponden. Si alguien se va, se anula su credencial una vez y pierde acceso a todo.

🔵 **En técnico:** Active Directory Domain Services es un **servicio de directorio** que guarda de forma centralizada usuarios, equipos, grupos y políticas, y los usa para **autenticar** (comprobar quién eres) y **autorizar** (decidir a qué accedes).

🎤 **En entrevista:** "AD DS centraliza las identidades de la empresa. Cada usuario tiene una sola cuenta para todos los equipos y recursos del dominio, y desde un solo punto se aplican políticas, permisos y altas o bajas."

### Las piezas de AD

| Concepto | 🟢 En simple | 🔵 En técnico |
|---|---|---|
| **Bosque** | El grupo empresarial completo | Contenedor máximo de AD y **límite de seguridad** real. Los dominios de un bosque confían entre sí. |
| **Dominio** | Una empresa del grupo | Límite de administración y replicación. Comparte base de datos, usuarios y políticas. |
| **Controlador de dominio (DC)** | La oficina de recursos humanos y seguridad que valida credenciales | Servidor con una copia de la base de AD que autentica a usuarios y equipos. |
| **OU** | Los departamentos dentro de la empresa | Contenedor para organizar objetos, aplicar GPO y delegar administración. |
| **Objeto** | Cada persona, PC o grupo de la lista | Cada entrada del directorio: usuario, equipo, grupo. |
| **Promover** | Convertir una oficina cualquiera en la de recursos humanos | Instalar AD DS y convertir un servidor en DC. El primero crea bosque, dominio y DC a la vez. |

---

## 2. NTDS.dit y SYSVOL

🟢 **En simple:** **NTDS.dit** es la **caja fuerte** con el archivo de todos los empleados y la "huella" de sus contraseñas. **SYSVOL** es el **tablón de anuncios** con las normas que todos los PCs deben cumplir.

🔵 **En técnico:**
- **NTDS.dit** (`C:\Windows\NTDS\`): la base de datos de AD. Contiene los objetos y los **hashes** de las contraseñas de todas las cuentas.
- **SYSVOL** (`C:\Windows\SYSVOL\`): carpeta compartida que se replica entre DCs. Contiene las **GPO** y los scripts de inicio de sesión.
- **Contraseña DSRM:** la contraseña del modo de restauración de AD, el "modo seguro" del DC.

### Pregunta 1 · Un atacante roba NTDS.dit

🟢 **En simple:** se llevó la caja fuerte con las huellas de **todas** las llaves de la empresa. Con ellas puede fabricar copias y entrar donde quiera, incluso como el dueño.

🔵 **En técnico:**
- Tiene los **hashes** de las contraseñas (no las contraseñas en texto). Puede **descifrarlos sin conexión** (*cracking offline*) o usarlos directamente para autenticarse (**Pass-the-Hash**).
- Lo más grave: obtiene el hash de la cuenta **krbtgt**, con la que se firman todos los tickets de Kerberos. Con ella puede crear un **Golden Ticket**: un ticket falso válido como cualquier usuario, incluso administrador, durante años.
- Es un **compromiso total del dominio**. La recuperación exige restablecer las contraseñas de todas las cuentas y la de krbtgt **dos veces**.

🎤 **En entrevista:** "NTDS.dit contiene los hashes de todas las cuentas del dominio. Con él, un atacante puede hacer Pass-the-Hash o crackearlos, y con el hash de krbtgt puede crear Golden Tickets. Es un compromiso total del dominio, por eso los DCs son activos Tier 0 y van en la zona más protegida."

---

## DNS desde cero (repaso fundamental)

### IP: quién soy

🟢 **En simple:** la IP es el **número de teléfono** de cada equipo. Para hablar con alguien en la red, siempre necesitas su número: los equipos se comunican por números, no por nombres.

🔵 **En técnico:** la dirección IP es una **dirección lógica** que identifica a un equipo dentro de una red. Los paquetes la usan para saber de dónde vienen y a dónde van.

### DNS: la agenda de contactos

🟢 **En simple:** las personas no recuerdan números. Buscas "Mamá" en la **agenda del celular** y el teléfono marca el número por ti. **El DNS es esa agenda**: escribes `dc01` y te da `10.0.10.10`.

🔵 **En técnico:** el DNS (*Domain Name System*) traduce **nombres a IPs** (zona directa) y **IPs a nombres** (zona inversa).

¿Para qué sirve, además de la comodidad?
1. **Las IPs pueden cambiar; los nombres no.** Si un servidor cambia de IP, se actualiza la agenda y todos lo siguen encontrando por su nombre.
2. **Active Directory depende de nombres.** Los equipos ubican al DC con registros **SRV**, y **Kerberos funciona con nombres**. Si accedes por IP, Windows suele usar **NTLM**, que es menos seguro.

### Ejemplo completo: María abre `\\dc01\contabilidad`

```
1. CLIENTE01: "¿Cuál es el número de dc01?"   → consulta al DNS (DC01)
2. DNS:       "dc01 es 10.0.10.10"           → responde desde su zona
3. CLIENTE01: se conecta a 10.0.10.10         → usa la IP para comunicarse
4. DC01:      pide el ticket de Kerberos y abre la carpeta
```

**El DNS da el número. La IP es la que se usa para hablar.**

### ⭐ Los dos campos de la tarjeta de red

> **IP address = quién soy. DNS server = a quién le pregunto.**

| Campo | Pregunta que responde | 🟢 En simple |
|---|---|---|
| **IP address** | ¿Cuál es **mi** número? | Mi número de celular |
| **DNS server** | ¿A **quién** le pregunto cuando necesito un número? | El número de la **operadora de información** |

⚠️ El campo DNS **no traduce nada**: solo guarda la dirección del servidor que traduce.

| | CLIENTE01 (María) | DC01 (la recepcionista) |
|---|---|---|
| **IP address** | `10.0.10.100` | `10.0.10.10` |
| **DNS server** | `10.0.10.10` → pregunta a DC01 | `10.0.10.10` → **se pregunta a sí mismo** |

**¿Por qué en DC01 coinciden?** Porque DC01 también se comporta como **servidor DNS**: cuando necesita buscar un nombre, se consulta a sí mismo, porque es el único que tiene la zona `ad.securecorp.com`. Además, DC01 **también es cliente** de su propio dominio: necesita encontrar sus propios registros SRV y registrar su nombre.

🎤 **En entrevista:** "La IP identifica a cada equipo; el DNS traduce nombres a IPs. En un dominio, todos los equipos, incluido el propio DC, usan al controlador de dominio como DNS, porque es el único que tiene la zona del dominio y los registros SRV. En el DC, el DNS preferido es su propia IP (u otro DC) y el loopback va como alternativo."

### `10.0.10.10` vs. `127.0.0.1`

| Dirección | 🟢 En simple | 🔵 En técnico |
|---|---|---|
| `10.0.10.10` | Su número **real**, el que conoce toda la red | IP asignada a la tarjeta de red |
| `127.0.0.1` | Decir **"yo"**: siempre apunta a uno mismo, sea cual sea su número | Dirección de **loopback**. Siempre apunta al propio equipo, sin depender de la IP configurada. |

Regla de Microsoft: en un DC, **primero la IP real** (o la de otro DC) y **el loopback como alternativo**, nunca primero. Con dos DCs, cada uno pregunta primero al otro y después a sí mismo.

### Caso práctico: ¿qué pasa si DC01 cambia su IP a `10.0.10.15`?

| Quién | Qué pasa |
|---|---|
| Usuarios que entran por **IP** (`\\10.0.10.10\...`) | ❌ Se rompe. Hay que corregir a mano cada acceso directo y script. |
| Usuarios que entran por **nombre** (`\\dc01\...`) | ✅ DC01 actualiza solo su registro A (actualización dinámica). **Pero** los clientes siguen preguntando al DNS viejo (`10.0.10.10`), así que hay que actualizarles el campo DNS. Se hace una vez desde el **DHCP**. |
| **DC01 mismo** | *(Observación propia, 15/11/2026)* Al cambiar la IP, **su campo DNS preferido sigue diciendo `10.0.10.10`**: los dos campos son independientes. Lo salva el alternativo **`127.0.0.1`**, que siempre apunta a sí mismo. **Por eso existe el loopback como alternativo.** |

Procedimiento correcto si un DC cambia de IP:
1. Cambiar la IP address.
2. Cambiar también su DNS preferido a la nueva IP.
3. `ipconfig /registerdns` para actualizar sus registros A y PTR.
4. Actualizar el DNS de los clientes desde el DHCP.
5. Revisar todo lo que use la IP a mano: accesos directos, scripts, reglas de firewall.

**Conclusión:** un controlador de dominio **no se cambia de IP** salvo que no quede más remedio. Por eso se planifica fija desde el día 1.

### ¿En qué orden busca Windows un nombre?

🟢 **En simple:** antes de llamar a la operadora, revisas primero tu memoria reciente y luego tu libreta personal.

| Orden | Dónde busca | 🔵 En técnico |
|---|---|---|
| 1 | Memoria reciente | **Caché DNS** del equipo (respuestas recientes, guardadas durante su TTL) |
| 2 | Libreta personal | Archivo **hosts**: `C:\Windows\System32\drivers\etc\hosts` |
| 3 | La operadora | El **servidor DNS** configurado en la tarjeta de red |

🔐 **Dato de seguridad:** el malware a veces **modifica el archivo hosts** para redirigir nombres legítimos (por ejemplo, el de un banco) hacia servidores falsos. Si un equipo "resuelve raro", revisa el archivo hosts y la caché.

### Comandos útiles de DNS

**Ver la configuración**

| Comando | Para qué sirve |
|---|---|
| `ipconfig /all` | Ver IP, máscara, gateway y **servidores DNS** configurados |
| `Get-DnsClientServerAddress` | Ver los DNS de cada tarjeta (PowerShell) |

**Consultar nombres**

| Comando | Para qué sirve |
|---|---|
| `nslookup dc01` | Resolver un nombre a IP (zona directa) |
| `nslookup 10.0.10.10` | Resolver una IP a nombre (zona inversa, PTR) |
| `nslookup -type=SRV _ldap._tcp.dc._msdcs.ad.securecorp.com` | Encontrar los controladores de dominio (registros SRV) |
| `nslookup google.com 8.8.8.8` | Preguntar a **otro servidor DNS** en concreto (útil para comparar) |
| `Resolve-DnsName dc01` | Equivalente moderno en PowerShell; devuelve objetos |
| `Resolve-DnsName _ldap._tcp.dc._msdcs.ad.securecorp.com -Type SRV` | SRV con PowerShell |

**Caché y registro**

| Comando | Para qué sirve |
|---|---|
| `ipconfig /displaydns` | Ver la caché DNS del equipo |
| `ipconfig /flushdns` | Borrar la caché (cuando un registro cambió y el equipo sigue usando el viejo) |
| `Clear-DnsClientCache` | Lo mismo, en PowerShell |
| `ipconfig /registerdns` | Pedir al equipo que vuelva a registrar su nombre (A y PTR) en el DNS |

**Diagnóstico**

| Comando | Para qué sirve |
|---|---|
| `dcdiag /test:dns` | Chequeo completo del DNS de un controlador de dominio |
| `Test-NetConnection dc01 -Port 389` | Comprobar que un nombre resuelve **y** que el puerto responde (LDAP) |
| `Test-NetConnection dc01 -Port 88` | Lo mismo para Kerberos |
| `dnsmgmt.msc` | Abrir la consola gráfica de DNS |

### Cómo leer la salida de `nslookup`

| Mensaje | 🟢 En simple | 🔵 Qué significa |
|---|---|---|
| `Server: UnKnown` | La operadora no se presentó con su nombre | No hay registro **PTR** para la IP del servidor DNS (falta la zona inversa). La consulta puede funcionar igual. |
| `Non-authoritative answer` | "Te lo dice alguien que lo leyó en otra agenda" | La respuesta viene de la **caché** o de un reenviador, no del dueño de la zona. Es normal para nombres de internet. |
| `Non-existent domain` (**NXDOMAIN**) | "Ese contacto no existe en la agenda" | El nombre no existe en esa zona, o le preguntaste al DNS equivocado (la trampa del 8.8.8.8) |
| `DNS request timed out` | La operadora no contesta | El servidor DNS no responde: está apagado, no es accesible o hay un firewall |

### Lista de diagnóstico: "no puedo acceder por nombre"

1. `ipconfig /all`: ¿el DNS configurado es el DC? ¿La IP es `169.254.x.x`? (Si lo es, el problema es DHCP, no DNS.)
2. `ping 10.0.10.10`: ¿hay conectividad con el DNS?
3. `nslookup <nombre>`: ¿resuelve? ¿Qué mensaje da?
4. `ipconfig /flushdns` y repetir, por si era una respuesta vieja en caché.
5. Revisar el archivo **hosts**.
6. En el DC: `dcdiag /test:dns` y la consola `dnsmgmt.msc`.

---

## 3. DNS en Active Directory

🟢 **En simple:** DNS es la **guía telefónica** de la red. Tú sabes el nombre ("dc01") y la guía te da el número (la IP). Sin guía, nadie encuentra a nadie.

🔵 **En técnico:** DNS traduce nombres a IPs y viceversa. AD **depende** de DNS: los equipos encuentran a los controladores de dominio consultando registros especiales (**SRV**) que el DC registra al ser promovido.

### Zonas DNS

| Zona | 🟢 En simple | 🔵 En técnico | Registro principal |
|---|---|---|---|
| **Directa** | Buscar el número sabiendo el nombre | Nombre → IP | **A** (IPv4), **AAAA** (IPv6). También viven aquí **SRV** y **CNAME**. |
| **Inversa** | Buscar el nombre sabiendo el número | IP → nombre | **PTR** |

**Cómo se nombra la zona inversa:** se toman solo los octetos de la **red** y se escriben **al revés**, seguidos de `.in-addr.arpa`.

| Qué es | Cómo se forma | Ejemplo |
|---|---|---|
| **Zona inversa** (la red completa) | Octetos de la red, al revés | `10.0.10.0/24` → **`10.0.10.in-addr.arpa`** (`10.0.10` es un palíndromo: al revés se lee igual) |
| **Registro PTR** (un equipo) | Los 4 octetos de la IP, al revés | DC01 (`10.0.10.10`) → **`10.10.0.10.in-addr.arpa`** |

Con otras redes se nota la inversión: `10.0.20.0/24` → `20.0.10.in-addr.arpa` · `192.168.1.0/24` → `1.168.192.in-addr.arpa`.

🔵 **¿Por qué al revés?** El DNS lee de derecha a izquierda, de lo general a lo específico (`.com` → `securecorp` → `ad`). En una IP lo general (la red) está a la izquierda. Al invertirla, queda organizada igual que los nombres.

> *Corrección (15/11/2026): en una versión anterior de esta nota decía que la zona se llamaba `10.10.0.10.in-addr.arpa`. Ese es el nombre del PTR de DC01, no el de la zona. El asistente de DNS generó el nombre correcto.*

### Registros

| Registro | 🟢 En simple | 🔵 En técnico | Ejemplo |
|---|---|---|---|
| **A** | "dc01 vive en esta dirección" | Nombre → IPv4 | `dc01` → `10.0.10.10` |
| **AAAA** | Lo mismo, con dirección nueva (IPv6) | Nombre → IPv6 | |
| **PTR** | "Esta dirección es la casa de dc01" | IPv4 → nombre (zona inversa) | `10.0.10.10` → `dc01.securecorp…` |
| **CNAME** | Un apodo | Alias de otro nombre | `intranet` → `dc01` |
| **SRV** | Un anuncio: "¿buscas recursos humanos? Ve a dc01, ventanilla 389" | Ubica un **servicio**: devuelve servidor y puerto | `_ldap._tcp.dc._msdcs.<dominio>` → `dc01:389` |

### Pregunta 3 · Zona directa vs. inversa
🎤 **En entrevista:** "La zona directa resuelve nombres a IPs con registros A y AAAA, y ahí también viven los SRV que usa AD. La zona inversa resuelve IPs a nombres con registros PTR. La inversa se usa para diagnóstico, registros de auditoría y algunas aplicaciones que verifican la identidad del equipo."

### Zona integrada en AD
🟢 **En simple:** la guía telefónica se guarda dentro de la caja fuerte de AD, y solo los empleados verificados pueden actualizar su propia entrada.

🔵 **En técnico:** la zona se almacena en la base de AD y se replica con ella. Permite **actualizaciones dinámicas seguras**: solo los equipos del dominio, autenticados, pueden registrar o cambiar sus propios registros. Evita que un intruso registre `dc01` apuntando a su máquina.

---

## Zonas DNS: repaso

### ¿Qué es una zona?

🟢 **En simple:** el DNS de todo internet es una **biblioteca gigante de guías telefónicas**. Nadie tiene la biblioteca completa: cada organización es dueña de **sus propios cuadernos**. Una **zona** es uno de esos cuadernos, y el servidor que es su dueño tiene la palabra final sobre lo que dice.

🔵 **En técnico:** una zona es una porción del espacio de nombres DNS administrada por un servidor **autoritativo** para ella: sus respuestas son oficiales, no copiadas de otro servidor.

Por eso `8.8.8.8` no sabe nada de SecureCorp: el cuaderno `ad.securecorp.com` solo existe en DC01.

### Directa vs. inversa

| | **Zona directa** | **Zona inversa** |
|---|---|---|
| 🟢 En simple | "Tengo el **nombre**, ¿cuál es su número?" | "Tengo un **número**, ¿de quién es?" |
| 🔵 En técnico | Nombre → IP | IP → nombre |
| Registro principal | **A** (y también SRV, CNAME) | **PTR** |
| ¿Quién la usa? | Todos, todo el tiempo, para conectarse | Diagnóstico, registros de seguridad, auditorías |
| ¿Obligatoria para AD? | ✅ Sí: contiene los registros SRV | ❌ No, pero es **buena práctica** |
| ¿Quién la crea? | La promoción, automáticamente | **Yo, a mano**, una por cada subred, porque solo yo conozco mis redes |

**Para qué sirve la zona inversa:**
- **Diagnóstico:** sin ella, `nslookup` muestra `Server: UnKnown` y tarda (*timeouts*). Lo vi en vivo el 15/11/2026, y desapareció al crearla.
- **Seguridad:** si un log dice *"intentos fallidos desde 10.0.20.137"*, la zona inversa dice **qué equipo es**. Clave para el Nivel 3 (evento 4625).
- **Aplicaciones** que verifican que nombre e IP coincidan en ambas direcciones.

### ⚠️ Una zona inversa por cada red

| Red | Zona inversa | Estado |
|---|---|---|
| `10.0.10.0/24` SERVIDORES | `10.0.10.in-addr.arpa` (palíndromo) | ✅ Creada |
| `10.0.20.0/24` USUARIOS | `20.0.10.in-addr.arpa` | Nivel 2 |
| `10.0.30.0/24` ADMIN | `30.0.10.in-addr.arpa` | Nivel 2 |
| `10.0.40.0/24` INVITADOS | `40.0.10.in-addr.arpa` | Nivel 2 (si se decide registrarlos) |

> **Lista de verificación para una subred nueva:** ámbito DHCP + zona inversa + reglas de firewall + documentación del plan IP.

### Zonas que tiene hoy DC01

| Zona | Tipo | Creada por | Qué guarda |
|---|---|---|---|
| `ad.securecorp.com` | Directa | La promoción | Registros A de los equipos y SRV del dominio |
| `_msdcs.ad.securecorp.com` | Directa | La promoción | SRV que ubican a los DCs de **todo el bosque** (la que consulta `nslookup -type=SRV _ldap._tcp.dc._msdcs…`) |
| `10.0.10.in-addr.arpa` | Inversa | Yo | PTR de DC01 y de los equipos que se registren |

### Registros vistos en la consola DNS

| Registro | 🟢 En simple | 🔵 En técnico |
|---|---|---|
| **SOA** | La **portada del cuaderno**: quién es el dueño y en qué versión va | *Start of Authority*: servidor principal de la zona y número de serie, que aumenta con cada cambio |
| **NS** | "Las recepcionistas oficiales de este cuaderno son…" | *Name Server*: servidores DNS autoritativos para la zona |
| **PTR** | El renglón número → nombre | Puntero de IP a nombre |

### ¿Qué significa "guardar la zona en Active Directory"?

No significa que la zona sea parte del dominio: indica **dónde y cómo se guarda el cuaderno**.

| | **Zona estándar** (archivo) | **Zona integrada en AD** ✅ |
|---|---|---|
| 🟢 En simple | Un **papel suelto en el escritorio** de la recepcionista | El cuaderno guardado **dentro de la caja fuerte** de AD |
| Dónde vive | Archivo de texto en `C:\Windows\System32\dns\` | Dentro de la base de datos de AD |
| Copias en otros servidores | Solo el **principal** escribe; los **secundarios** tienen fotocopias que se actualizan por **transferencia de zona** | Se replica con AD. **Multimaestro:** cualquier DC puede escribir. |
| Si cae el servidor principal | No se pueden hacer cambios | Otro DC sigue funcionando y editando |
| Quién actualiza registros | Cualquiera, o nadie | Solo equipos del dominio autenticados (**actualizaciones dinámicas seguras**) |

**Las 3 ventajas:**
1. **Se replica sola:** cuando llegue DC02, recibirá la zona automáticamente con AD, sin configurar transferencias de zona.
2. **Tolerancia a fallos:** cualquier DC puede seguir respondiendo **y editando** la zona.
3. **Seguridad:** las actualizaciones dinámicas seguras exigen Kerberos e impiden que un atacante registre `dc01` apuntando a su máquina (**DNS spoofing**).

🔐 Como la zona vive dentro de AD, proteger los DCs protege el DNS. Y si alguien compromete AD, también controla el DNS. Otro motivo para que los DCs sean **Tier 0**.

🎤 **En entrevista:** "Una zona es la parte del espacio DNS de la que un servidor es autoritativo. La directa resuelve nombres a IPs y es obligatoria para AD porque contiene los SRV; la crea la promoción. La inversa resuelve IPs a nombres, se crea a mano por cada subred y es clave para diagnóstico y auditoría. Las integro en AD porque se replican con AD, son multimaestro y permiten actualizaciones dinámicas seguras."

### Preguntas de repaso

**1. "La zona inversa es obligatoria; sin ella el dominio no funciona." ¿Es cierto?**
No. AD solo necesita la zona directa, donde están los registros SRV. La inversa es buena práctica, sobre todo para seguridad y diagnóstico.

**2. ¿Qué ventaja habrá con DC02 gracias a que la zona está integrada en AD?**
DC02 recibirá la zona automáticamente como parte de la replicación de AD, sin configurar transferencias de zona. Y como es multimaestro, también podrá modificarla.

**3. Un log dice "10 intentos fallidos desde 10.0.20.137". ¿Qué zona y qué registro dicen de qué equipo se trata?**
El registro **PTR** de esa IP, en la zona inversa de su subred: **`20.0.10.in-addr.arpa`** (red USUARIOS). ⚠️ No es `10.0.10.in-addr.arpa`: hay una zona inversa por cada red, y la de servidores solo parece "sin invertir" porque `10.0.10` es un palíndromo.

---

## 4. Servidor DNS vs. reenviador (⚠️ repasar)

🟢 **En simple:**
- **DC01 es la recepcionista** de SecureCorp. Sabe dónde está cada oficina interna (`securecorp`).
- Si le preguntas por una dirección de afuera (`google.com`), ella **llama a información** (el **reenviador**) y te pasa la respuesta.
- Los empleados **siempre le preguntan a la recepcionista**. Si un empleado llama directo a información (`8.8.8.8`), le dirán que no conocen ninguna oficina de SecureCorp.

🔵 **En técnico:**

| | Servidor DNS del dominio | Reenviador (*forwarder*) |
|---|---|---|
| ¿Quién es? | DC01 (`10.0.10.10`) | Un DNS externo (`1.1.1.1`, `8.8.8.8`) |
| ¿Qué sabe? | La zona del dominio y sus registros SRV | Nombres públicos de internet |
| ¿Dónde se configura? | En la **tarjeta de red** de clientes y servidores | **Dentro del servicio DNS de DC01**, nunca en los clientes |
| Dirección de la consulta | Cliente → DC01 | DC01 → reenviador (solo para lo que DC01 no conoce) |

### Pregunta 2 · CLIENTE01 usa 8.8.8.8 e intenta unirse al dominio

🟢 **En simple:** CLIENTE01 llama directamente a información pública y pregunta por la oficina de recursos humanos de SecureCorp. Información no conoce nada interno de SecureCorp y responde "no existe". CLIENTE01 nunca encuentra al DC. **Aquí no hay ningún reenvío**: la recepcionista ni se enteró de la llamada.

🔵 **En técnico:**
1. CLIENTE01 consulta **directamente** a `8.8.8.8` por el registro SRV `_ldap._tcp.dc._msdcs.<dominio>`.
2. Google no tiene esa zona, porque es interna, y responde **NXDOMAIN** (el nombre no existe).
3. Sin el SRV, CLIENTE01 no sabe qué servidor es el DC ni en qué puerto contactarlo.
4. Error que ve el usuario: *"An Active Directory Domain Controller (AD DC) for the domain could not be contacted"*.
5. Si un equipo ya unido cambia a `8.8.8.8`: falla el inicio de sesión con cuentas del dominio, no se aplican GPO y Kerberos no funciona.

🎤 **En entrevista:** "El cliente busca el registro SRV del controlador de dominio. Si su DNS es 8.8.8.8, la consulta va directo a Google, que no tiene la zona interna y responde que no existe. No puede ubicar al DC y no se puede unir al dominio. Los clientes deben usar solo los DCs como DNS; el DNS público se configura como reenviador dentro del DC."

---

## 5. Kerberos

🟢 **En simple:** funciona como un **parque de diversiones**:
1. Muestras tu DNI en la boletería **una sola vez**.
2. Te dan una **pulsera** válida por el día.
3. En cada juego muestras la pulsera y te dan una **entrada** para ese juego.
4. Entras sin volver a mostrar el DNI.

Tu contraseña es el DNI: **nunca la entregas a cada juego**, ni siquiera viaja por la red.

🔵 **En técnico:**

| Parque | Kerberos |
|---|---|
| Boletería | **KDC** (*Key Distribution Center*), un servicio que corre en cada DC |
| Mostrar el DNI | El cliente envía datos cifrados con una clave derivada de su contraseña (**AS-REQ**). La contraseña no viaja. |
| La pulsera | **TGT** (*Ticket Granting Ticket*): cifrado con la clave de **krbtgt**, válido 10 horas por defecto |
| La entrada a un juego | **Ticket de servicio** (TGS) para un recurso concreto (carpeta, impresora, etc.) |
| Entrar sin DNI | **SSO** (*Single Sign-On*, inicio de sesión único) |

Datos clave:
- **Puerto 88** (TCP/UDP).
- **Funciona con nombres, no con IPs.** Si accedes por IP (`\\10.0.10.10\carpeta`), Windows suele usar **NTLM**, un protocolo más antiguo y débil.
- **Tolerancia de hora: 5 minutos.**

### Pregunta 4 · El reloj está adelantado 8 minutos

🟢 **En simple:** el portero revisa la hora impresa en tu entrada. Si dice una hora muy distinta a la real, sospecha que es una entrada vieja o copiada y no te deja pasar, aunque sea auténtica.

🔵 **En técnico:** cada solicitud de Kerberos lleva una **marca de tiempo** cifrada. Si la diferencia entre el cliente y el DC supera los **5 minutos** (valor por defecto), el KDC la rechaza (error `KRB_AP_ERR_SKEW`). Así se impiden los **ataques de repetición** (*replay*): que alguien capture una solicitud y la reenvíe más tarde. En un dominio, el DC es la fuente de hora de todos los equipos.

🎤 **En entrevista:** "Kerberos rechaza la autenticación si la hora del cliente difiere en más de 5 minutos de la del DC, para evitar ataques de repetición. Por eso la sincronización de hora es crítica en un dominio: el DC es la fuente de hora de todos los equipos."

### Pregunta 5 · ¿Qué es un TGT?

🟢 **En simple:** es la **pulsera** del parque. Prueba que ya mostraste tu DNI en la boletería, y la usas para pedir entradas a cada juego sin volver a identificarte.

🔵 **En técnico:** el **TGT** es el ticket que entrega el **KDC** cuando el usuario se autentica al iniciar sesión. Está cifrado con la clave de la cuenta **krbtgt** y dura 10 horas por defecto. Con él, el cliente pide **tickets de servicio** para cada recurso sin volver a usar su contraseña. Eso es lo que hace posible el **SSO**.

🎤 **En entrevista:** "El TGT es el ticket que el KDC entrega al iniciar sesión. Con él se piden tickets de servicio para cada recurso sin volver a usar la contraseña: eso es el SSO. Está firmado con la clave de krbtgt; por eso, si esa clave se roba, se pueden falsificar TGTs, el ataque llamado Golden Ticket."

### 🔐 Para la etapa de seguridad (solo para reconocer los nombres)

| Ataque | 🟢 En simple | Qué abusa |
|---|---|---|
| **Pass-the-Hash** | Usar la huella de la llave sin conocer la llave | Los hashes de NTLM |
| **Kerberoasting** | Pedir entradas de juegos y descifrarlas en casa para sacar la clave del juego | Tickets de servicio con contraseñas débiles |
| **Golden Ticket** | Fabricar pulseras falsas con la máquina de la boletería | El hash de krbtgt |

---

## 6. El nombre del dominio

### Pregunta 6 · Decisión: `ad.securecorp.com`

| Opción | 🟢 En simple | 🔵 En técnico |
|---|---|---|
| `securecorp.local` | Un nombre inventado que choca con otros aparatos de la casa | `.local` está reservado para **mDNS** (Bonjour, impresoras, Linux). Causa conflictos y no admite certificados públicos. Microsoft ya no lo recomienda. |
| `securecorp.com` | Usar el mismo nombre dentro y fuera, y mantener dos guías telefónicas a mano | **Split-brain DNS**: hay que replicar a mano en el DNS interno los registros públicos (web, correo). |
| **`ad.securecorp.com`** ✅ | Un "anexo" interno del nombre oficial de la empresa | **Subdominio de un dominio propio.** Sin conflictos, admite certificados y está listo para la integración con Entra ID. Es la recomendación de Microsoft. |

**Mi decisión:** `ad.securecorp.com`, porque es la práctica recomendada por Microsoft, evita los conflictos de `.local` y facilita la integración híbrida con Azure (Nivel 4).

### ⚠️ Detalle práctico: el nombre NetBIOS

🟢 **En simple:** además del nombre completo, el dominio tiene un **nombre corto** que los usuarios escriben al iniciar sesión.

🔵 **En técnico:** el **nombre NetBIOS** (máximo 15 caracteres) se usa en el formato de inicio de sesión `DOMINIO\usuario`. El asistente propone la **primera parte** del nombre completo. Con `ad.securecorp.com` propondría **`AD`**, y los usuarios iniciarían sesión como `AD\maria.lopez`, lo que confunde. **Hay que cambiarlo a `SECURECORP`** durante la promoción, y **no se puede cambiar fácilmente después**.

| Formato de inicio de sesión | Ejemplo |
|---|---|
| NetBIOS | `SECURECORP\maria.lopez` |
| UPN (estilo correo) | `maria.lopez@ad.securecorp.com` |

---

## 7. Resumen en una tabla

| Concepto | 🟢 En simple | 🔵 En técnico |
|---|---|---|
| AD DS | Lista oficial de personal | Servicio de directorio centralizado |
| DC | Recursos humanos y seguridad | Servidor que guarda AD y autentica |
| NTDS.dit | La caja fuerte | Base de datos de AD con los hashes |
| SYSVOL | Tablón de normas | Carpeta replicada con GPO y scripts |
| DNS | Guía telefónica | Resolución de nombres ↔ IPs |
| Registro SRV | Anuncio "ve a tal ventanilla" | Ubica servicios: servidor + puerto |
| Reenviador | La recepcionista llamando a información | DNS externo que consulta el DC para nombres públicos |
| Kerberos | Parque de diversiones | Autenticación por tickets, puerto 88 |
| KDC | La boletería | Servicio del DC que emite tickets |
| TGT | La pulsera | Ticket inicial firmado con krbtgt, 10 h |
| Ticket de servicio | La entrada a un juego | Ticket para un recurso concreto |
| SSO | Entrar a todo con una sola identificación | Inicio de sesión único |
| NetBIOS | Nombre corto del dominio | Nombre de hasta 15 caracteres para `DOMINIO\usuario` |
