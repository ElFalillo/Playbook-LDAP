# Playbook de Despliegue Técnico
## Instalación, configuración y puesta en marcha de OpenLDAP

**Alumno:** Raul Ortiz  
**Entorno:** Ubuntu Server 24.04.5 LTS sobre Oracle VirtualBox  
**Servicio desplegado:** OpenLDAP  
**Fecha de realización:** 09/10/2026  

---

## A. Ficha Técnica y Prerrequisitos del Entorno

### A.1. Resumen técnico

| Elemento | Configuración aplicada |
|---|---|
| Hipervisor | Oracle VirtualBox |
| Sistema operativo | Ubuntu Server 24.04.5 LTS |
| Nombre del equipo | `UbuntuServer` |
| Tipo de red | Red NAT sin DHCP |
| Interfaz de red | `enp0s3` |
| Dirección IP estática | `20.0.0.5/24` |
| Puerta de enlace | `20.0.0.1` |
| DNS configurados | `20.0.0.1` y `8.8.8.8` |
| FQDN | `ldapserver.planetafp.local` |
| Alias | `ldapserver` |
| Dominio LDAP | `planetafp.local` |
| Base DN | `dc=planetafp,dc=local` |
| Organización | `planetafp` |
| Cuenta administradora | `cn=admin,dc=planetafp,dc=local` |
| Paquetes instalados | `slapd`, `ldap-utils`, `libldap-common`, `libldap2` |
| Puerto LDAP | `389/tcp` |
| Versión OpenLDAP | `2.6.10+dfsg-0ubuntu0.24.04.1` |

### A.2. Servicio y componentes

OpenLDAP es una implementación de código abierto del protocolo LDAP (*Lightweight Directory Access Protocol*). Permite guardar y consultar información organizada de forma jerárquica, por ejemplo usuarios, unidades organizativas, grupos y atributos de autenticación.

Los componentes principales desplegados son:

- **`slapd`**: daemon o servicio servidor LDAP; recibe peticiones de los clientes y gestiona el directorio.
- **`ldap-utils`**: herramientas de línea de comandos para administrar el directorio, como `ldapadd`, `ldapsearch`, `ldapdelete`, `ldapmodify` y `ldapwhoami`.
- **`libldap`**: bibliotecas utilizadas por aplicaciones y utilidades cliente para comunicarse con el servicio LDAP.
- **Esquemas LDAP**: definen las clases de objeto y los atributos admitidos por las entradas del directorio.

### A.3. Evidencia del sistema operativo

Se comprobó la versión del sistema operativo antes de iniciar el despliegue.

```bash
lsb_release -a
```

![Figura 1 — Información del sistema Ubuntu Server](Captura-de-pantalla-2026-10-09-183525.jpg)

*Figura 1. Ubuntu Server 24.04.5 LTS, nombre en clave `noble`.*

### A.4. Evidencia de paquetes instalados

Se verificó que los paquetes requeridos para el servicio LDAP estaban instalados.

```bash
dpkg -l | grep -E "slapd|ldap-utils|libldap"
```

![Figura 2 — Paquetes LDAP instalados](./Captura-de-pantalla-2026-10-09-183539.jpg)

*Figura 2. Paquetes `slapd`, `ldap-utils`, `libldap-common` y `libldap2` instalados.*

---

## B. Procedimiento de Despliegue Paso a Paso

### Paso 1. Configuración de red estática con Netplan

La máquina virtual se configuró con Red NAT sin DHCP. Para mantener una dirección permanente para el servidor LDAP se asignó la IP estática `20.0.0.5/24`.

El archivo configurado fue:

```text
/etc/netplan/50-cloud-init.yaml
```

Configuración aplicada:

```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: false
      addresses:
        - 20.0.0.5/24
      routes:
        - to: default
          via: 20.0.0.1
      nameservers:
        addresses:
          - 20.0.0.1
          - 8.8.8.8
```

Después de guardar el fichero, se aplicó la configuración:

```bash
sudo netplan apply
```

#### Evidencia de Netplan

![Figura 3 — Archivo de configuración Netplan](./Captura-de-pantalla-2026-10-09-183551.jpg)

*Figura 3. Configuración estática de la interfaz `enp0s3`: IP `20.0.0.5/24`, gateway `20.0.0.1` y DNS configurados.*

#### Verificación de dirección IP

```bash
ip a
```

![Figura 4 — Dirección IP estática](./Captura-de-pantalla-2026-10-09-183646.jpg)

*Figura 4. La interfaz `enp0s3` tiene asignada la IP estática `20.0.0.5/24`.*

#### Verificación de rutas

```bash
ip route
```

![Figura 5 — Tabla de enrutamiento](./Captura-de-pantalla-2026-10-09-183714.jpg)

*Figura 5. La ruta por defecto utiliza la puerta de enlace `20.0.0.1` a través de la interfaz `enp0s3`.*

---

### Paso 2. Configuración de resolución de nombres local

Al no disponer de un DNS interno para el dominio `planetafp.local`, se configuró la resolución local mediante el fichero `/etc/hosts`.

Se añadió la siguiente entrada:

```text
20.0.0.5    ldapserver.planetafp.local ldapserver
```

Esta entrada permite utilizar tanto el FQDN como el alias corto para localizar el servidor LDAP sin necesitar un servidor DNS externo.

#### Evidencia de `/etc/hosts`

```bash
cat /etc/hosts
```

![Figura 6 — Resolución local configurada](./Captura-de-pantalla-2026-10-09-183749.jpg)

*Figura 6. Asociación de la IP `20.0.0.5` con el FQDN `ldapserver.planetafp.local` y el alias `ldapserver`.*

#### Verificación de FQDN y alias

```bash
ping ldapserver.planetafp.local
ping ldapserver
```

![Figura 7 — Comprobación de resolución local](./Captura-de-pantalla-2026-10-09-183803.jpg)

*Figura 7. El FQDN y el alias resuelven correctamente a la dirección `20.0.0.5`.*

---

### Paso 3. Actualización de repositorios e instalación

Se actualizaron primero los repositorios del sistema:

```bash
sudo apt update
```

![Figura 8 — Actualización de repositorios](./Captura-de-pantalla-2026-10-09-183821.jpg)

*Figura 8. Actualización correcta de los índices de paquetes de Ubuntu.*

A continuación, se instalaron el servicio OpenLDAP y sus herramientas de administración:

```bash
sudo apt install -y slapd ldap-utils libldap-common
```

![Figura 9 — Instalación de OpenLDAP](./Captura-de-pantalla-2026-10-09-183857.jpg)

*Figura 9. Confirmación de que `slapd`, `ldap-utils` y `libldap-common` están instalados y actualizados.*

---

### Paso 4. Reconfiguración del servicio `slapd`

Se ejecutó el asistente de configuración de OpenLDAP:

```bash
sudo dpkg-reconfigure slapd
```

Las opciones configuradas fueron las siguientes:

| Pregunta del asistente | Respuesta seleccionada |
|---|---|
| Omitir la configuración del servidor OpenLDAP | No |
| Nombre DNS del dominio | `planetafp.local` |
| Nombre de la organización | `planetafp` |
| Contraseña del administrador LDAP | Configurada durante la práctica |
| Motor de base de datos | MDB |
| Eliminar la base de datos al purgar `slapd` | No |
| Mover la base de datos anterior | Sí |
| Permitir LDAPv2 | No |

El dominio LDAP definido da lugar a la siguiente base DN:

```text
dc=planetafp,dc=local
```

La cuenta de administración del directorio es:

```text
cn=admin,dc=planetafp,dc=local
```

#### Dominio LDAP

![Figura 10 — Dominio LDAP configurado](./Captura-de-pantalla-2026-10-09-184056.jpg)

*Figura 10. Configuración del nombre de dominio DNS `planetafp.local`.*

#### Organización LDAP

![Figura 11 — Organización configurada](./Captura-de-pantalla-2026-10-09-184048.jpg)

*Figura 11. Configuración de la organización `planetafp`.*

#### Conservación de la base de datos

![Figura 12 — No eliminar la base de datos al purgar](./Captura-de-pantalla-2026-10-09-184121.jpg)

*Figura 12. Se seleccionó la opción `No` para conservar la base de datos si `slapd` se purga.*

#### Movimiento de la base anterior

![Figura 13 — Mover base de datos antigua](./Captura-de-pantalla-2026-10-09-184401.jpg)

*Figura 13. Se seleccionó `Yes` para mover los archivos de una base de datos anterior antes de crear una nueva.*

---

## C. Verificación y Comprobación

### C.1. Estado del servicio LDAP

Se comprobó que el daemon OpenLDAP se hubiera iniciado correctamente:

```bash
sudo systemctl status slapd
```

![Figura 14 — Estado activo de slapd](./Captura-de-pantalla-2026-10-09-184421.jpg)

*Figura 14. El servicio `slapd` se encuentra en estado `active (running)`.*

### C.2. Puerto LDAP en escucha

Se verificó que el servicio estuviera escuchando en el puerto LDAP estándar, TCP 389:

```bash
sudo ss -lntp | grep 389
```

![Figura 15 — Puerto LDAP 389 en escucha](./Captura-de-pantalla-2026-10-09-185217.jpg)

*Figura 15. El proceso `slapd` escucha en el puerto `389` para IPv4 (`0.0.0.0:389`) e IPv6 (`[::]:389`).*

### C.3. Verificación de la base DN

Se consultó el contenido de la base LDAP:

```bash
ldapsearch -x -LLL -b dc=planetafp,dc=local
```

![Figura 16 — Base DN de OpenLDAP](./Captura-de-pantalla-2026-10-09-185249.jpg)

*Figura 16. La base del directorio LDAP contiene la entrada raíz `dc=planetafp,dc=local`, con la organización `planetafp`.*

### C.4. Esquemas LDAP disponibles

Se verificaron los esquemas disponibles en el sistema:

```bash
ls -l /etc/ldap/schema/
```

![Figura 17 — Esquemas disponibles](./Captura-de-pantalla-2026-10-09-185320.jpg)

*Figura 17. Esquemas LDAP disponibles, incluyendo `core`, `cosine`, `inetorgperson`, `nis` y otros.*

---

## D. Población del Directorio y Operaciones Básicas

### D.1. Creación del fichero LDIF

Se creó el fichero `estructura-inicial.ldif` para generar una estructura inicial en el directorio.

El fichero crea:

- La unidad organizativa `ou=personas`.
- El usuario `uid=raul`.
- El usuario se basa en las clases `inetOrgPerson` y `posixAccount`.

```ldif
dn: ou=personas,dc=planetafp,dc=local
objectClass: organizationalUnit
ou: personas

dn: uid=raul,ou=personas,dc=planetafp,dc=local
objectClass: inetOrgPerson
objectClass: posixAccount
uid: raul
cn: Raul Ortiz
sn: Ortiz
givenName: Raul
mail: raul@planetafp.local
uidNumber: 10001
gidNumber: 10001
homeDirectory: /home/raul
userPassword: {SSHA}CambiarEstaContrasena
```

> La contraseña indicada en el LDIF es un valor de prueba. En un entorno de producción, debe utilizarse un hash generado con `slappasswd` y nunca una contraseña en texto plano.

![Figura 18 — Fichero LDIF inicial](./Captura-de-pantalla-2026-10-09-185306.jpg)

*Figura 18. Contenido del fichero `estructura-inicial.ldif` utilizado para crear la OU y el usuario de prueba.*

### D.2. Inserción de entradas mediante `ldapadd`

Se cargó el fichero LDIF usando autenticación simple y bind autenticado como administrador:

```bash
ldapadd -x -D "cn=admin,dc=planetafp,dc=local" -W -f estructura-inicial.ldif
```

Significado de los parámetros:

| Parámetro | Descripción |
|---|---|
| `-x` | Utiliza autenticación simple |
| `-D` | Especifica el DN de la cuenta que realiza el bind |
| `-W` | Solicita la contraseña de forma interactiva |
| `-f` | Carga los datos desde el fichero LDIF indicado |

La operación creó correctamente la OU y el usuario LDAP.

![Figura 19 — Inserción correcta con ldapadd](./Captura-de-pantalla-2026-10-09-185530.jpg)

*Figura 19. Alta correcta de `ou=personas` y `uid=raul` mediante `ldapadd`.*

### D.3. Búsqueda por identificador de usuario

Se realizó una búsqueda delimitando la base, el ámbito y el filtro por UID:

```bash
ldapsearch -x -LLL \
  -b "ou=personas,dc=planetafp,dc=local" \
  -s sub \
  "(uid=raul)"
```

![Figura 20 — Búsqueda LDAP por uid](./Captura-de-pantalla-2026-10-09-185858.jpg)

*Figura 20. Resultado de la consulta LDAP para el usuario cuyo atributo `uid` es `raul`.*

### D.4. Búsqueda por clase de objeto

También se realizó una búsqueda sobre toda la base del directorio para recuperar las entradas de tipo `inetOrgPerson`:

```bash
ldapsearch -x -LLL \
  -b "dc=planetafp,dc=local" \
  -s sub \
  "(objectClass=inetOrgPerson)"
```

![Figura 21 — Búsqueda por objectClass](./Captura-de-pantalla-2026-10-09-185949.jpg)

*Figura 21. Consulta LDAP filtrada por la clase de objeto `inetOrgPerson`.*

### D.5. Verificación de identidad del administrador

Se comprobó el bind autenticado del administrador LDAP:

```bash
ldapwhoami -x -D "cn=admin,dc=planetafp,dc=local" -W
```

![Figura 22 — Bind correcto del administrador](./Captura-de-pantalla-2026-10-09-190041.jpg)

*Figura 22. El comando `ldapwhoami` confirma que la identidad autenticada es `cn=admin,dc=planetafp,dc=local`.*

---

## E. Control de Errores y Troubleshooting

### E.1. Error: entrada duplicada `Already exists (68)`

Durante una prueba se intentó volver a crear la entrada:

```text
uid=prueba,ou=personas,dc=planetafp,dc=local
```

El comando utilizado fue:

```bash
ldapadd -x -D "cn=admin,dc=planetafp,dc=local" -W -f error-sintaxis.ldif
```

La operación devolvió:

```text
ldap_add: Already exists (68)
```

El error indica que el DN solicitado ya existe en el directorio. LDAP no permite crear dos entradas con el mismo DN.

![Figura 23 — Error de entrada duplicada](./Captura-de-pantalla-2026-10-09-185530.jpg)

> **Nota:** sustituye la ruta de esta imagen por la captura exacta donde aparezca `uid=prueba` y `ldap_add: Already exists (68)`. En tu lista, esta es la captura tomada a las `19:00:41` o la que se corresponda visualmente con el error. Si usas la captura subida como primera imagen del chat, nómbrala `error-already-exists.jpg` y emplea esta ruta:

```markdown

```

**Causa:** la entrada ya había sido insertada con anterioridad.

**Diagnóstico recomendado:**

```bash
ldapsearch -x -LLL \
  -b "ou=personas,dc=planetafp,dc=local" \
  "(uid=prueba)"
```

**Solución posible 1: modificar la entrada existente**

Para cambiar atributos de una entrada existente se debe utilizar `ldapmodify`, no `ldapadd`.

**Solución posible 2: eliminar y volver a crear**

```bash
ldapdelete -x -D "cn=admin,dc=planetafp,dc=local" -W \
  "uid=prueba,ou=personas,dc=planetafp,dc=local"
```

Después se puede ejecutar de nuevo `ldapadd` con el LDIF correspondiente.

---

### E.2. Error: credenciales incorrectas `Invalid credentials (49)`

En una operación de inserción se produjo el siguiente error:

```bash
ldapadd -x -D "cn=admin,dc=planetafp,dc=local" -W -f estructura-inicial.ldif
```

Salida obtenida:

```text
ldap_bind: Invalid credentials (49)
```

El error se produjo porque se introdujo una contraseña incorrecta para el administrador LDAP.

![Figura 24 — Error de credenciales LDAP](./Captura-de-pantalla-2026-10-09-185858.jpg)

> **Nota:** sustituye la ruta de esta imagen por la captura donde aparezca literalmente `ldap_bind: Invalid credentials (49)`. Renómbrala como `error-invalid-credentials.jpg` y usa:

```markdown

```

**Diagnóstico y comprobación:**

Para comprobar si el DN es correcto y validar las credenciales se ejecutó:

```bash
ldapwhoami -x -D "cn=admin,dc=planetafp,dc=local" -W
```

La salida correcta fue:

```text
dn:cn=admin,dc=planetafp,dc=local
```

Esto confirmó que la cuenta administradora y el servicio estaban configurados correctamente; el problema se solucionó introduciendo la contraseña correcta.

![Figura 25 — Credenciales corregidas](./Captura-de-pantalla-2026-10-09-190041.jpg)

*Figura 25. Verificación correcta de la identidad del administrador con `ldapwhoami`.*

---

### E.3. Error: servidor LDAP no accesible

En una prueba con el servicio `slapd` detenido se ejecutó:

```bash
ldapsearch -x -LLL -b dc=planetafp,dc=local
```

La salida fue:

```text
ldap_sasl_bind(SIMPLE): Can't contact LDAP server (-1)
```

Este error se produce cuando el servicio LDAP está detenido o no está disponible en el puerto 389.

![Figura 26 — Error de contacto con el servidor LDAP](./Captura-de-pantalla-2026-10-09-185949.jpg)

> **Nota:** sustituye la ruta de esta imagen por la captura que muestra `Can't contact LDAP server (-1)`. Renómbrala como `error-cant-contact-ldap.jpg` y usa:

```markdown

```

**Solución aplicada:**

```bash
sudo systemctl start slapd
```

**Verificación posterior:**

```bash
sudo systemctl status slapd
sudo ss -lntp | grep 389
ldapsearch -x -LLL -b dc=planetafp,dc=local
```

La recuperación queda demostrada en las figuras 14, 15 y 16, donde el servicio aparece en estado `active (running)`, el puerto 389 está en escucha y la consulta LDAP devuelve la entrada raíz del directorio.

---

## Conclusión

Se ha desplegado y configurado correctamente un servidor OpenLDAP en Ubuntu Server 24.04.5 LTS dentro de una máquina virtual VirtualBox. El servidor tiene configurada la IP estática `20.0.0.5/24`, resolución local para `ldapserver.planetafp.local` y una base LDAP correspondiente al dominio `planetafp.local`.

La base `dc=planetafp,dc=local` se verificó correctamente y fue poblada mediante un archivo LDIF con la unidad organizativa `ou=personas` y el usuario `uid=raul`. Las operaciones de inserción, búsqueda y autenticación se realizaron satisfactoriamente con `ldapadd`, `ldapsearch` y `ldapwhoami`.

También se documentaron errores de entradas duplicadas, credenciales incorrectas y falta de conectividad con el servidor LDAP, indicando su causa, diagnóstico y solución.
