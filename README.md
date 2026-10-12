# Playbook de Despliegue Técnico
## Instalación, configuración y puesta en marcha de OpenLDAP

**Autores:** Falillo y JAM  
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

![Figura 1 — Información del sistema Ubuntu Server](capturas/Captura%20de%20pantalla%202026-10-09%20183525.png)

*Figura 1. Ubuntu Server 24.04.5 LTS, nombre en clave `noble`.*

### A.4. Evidencia de paquetes instalados

Se verificó que los paquetes requeridos para el servicio LDAP estaban instalados.

```bash
dpkg -l | grep -E "slapd|ldap-utils|libldap"
```

![Figura 2 — Paquetes LDAP instalados](capturas/Captura%20de%20pantalla%202026-10-09%20183539.png)

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

![Figura 3 — Archivo de configuración Netplan](capturas/Captura%20de%20pantalla%202026-10-09%20183551.png)

*Figura 3. Configuración estática de la interfaz `enp0s3`: IP `20.0.0.5/24`, gateway `20.0.0.1` y DNS configurados.*

#### Verificación de dirección IP

```bash
ip a
```

![Figura 4 — Dirección IP estática](capturas/Captura%20de%20pantalla%202026-10-09%20183646.png)

*Figura 4. La interfaz `enp0s3` tiene asignada la IP estática `20.0.0.5/24`.*

#### Verificación de rutas

```bash
ip route
```

![Figura 5 — Tabla de enrutamiento](capturas/Captura%20de%20pantalla%202026-10-09%20183714.png)

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

![Figura 6 — Resolución local configurada](capturas/Captura%20de%20pantalla%202026-10-09%20183749.png)

*Figura 6. Asociación de la IP `20.0.0.5` con el FQDN `ldapserver.planetafp.local` y el alias `ldapserver`.*

#### Verificación de FQDN y alias

```bash
ping ldapserver.planetafp.local
ping ldapserver
```

![Figura 7 — Comprobación de resolución local](capturas/Captura%20de%20pantalla%202026-10-09%20183803.png)

*Figura 7. El FQDN y el alias resuelven correctamente a la dirección `20.0.0.5`.*

---

### Paso 3. Actualización de repositorios e instalación

Se actualizaron primero los repositorios del sistema:

```bash
sudo apt update
```

![Figura 8 — Actualización de repositorios](capturas/Captura%20de%20pantalla%202026-10-09%20183821.png)

*Figura 8. Actualización de los índices de paquetes de Ubuntu.*

A continuación, se instalaron el servicio OpenLDAP y sus herramientas de administración:

```bash
sudo apt install -y slapd ldap-utils libldap-common
```

![Figura 9 — Instalación de OpenLDAP](capturas/Captura%20de%20pantalla%202026-10-09%20183857.png)

*Figura 9. Instalación de `slapd`, `ldap-utils` y `libldap-common`.*

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

#### Organización configurada

![Figura 10 — Organización LDAP](capturas/Captura%20de%20pantalla%202026-10-09%20184048.png)

*Figura 10. Configuración de la organización `planetafp`.*

#### Dominio LDAP

![Figura 11 — Dominio LDAP configurado](capturas/Captura%20de%20pantalla%202026-10-09%20184056.png)

*Figura 11. Configuración del nombre de dominio DNS `planetafp.local`.*

#### Conservación de la base de datos

![Figura 12 — Conservación de la base de datos](capturas/Captura%20de%20pantalla%202026-10-09%20184121.png)

*Figura 12. Opción seleccionada para conservar la base de datos si se purga `slapd`.*

#### Movimiento de la base anterior

![Figura 13 — Movimiento de base de datos anterior](capturas/Captura%20de%20pantalla%202026-10-09%20184401.png)

*Figura 13. Selección de la opción para mover los archivos de una base de datos anterior.*

---

## C. Verificación y Comprobación

### C.1. Estado del servicio LDAP

Se comprobó que el daemon OpenLDAP se hubiera iniciado correctamente:

```bash
sudo systemctl status slapd
```

![Figura 14 — Estado de slapd](capturas/Captura%20de%20pantalla%202026-10-09%20184421.png)

*Figura 14. Comprobación del estado del servicio `slapd`.*

### C.2. Puerto LDAP en escucha

Se verificó que el servicio estuviera escuchando en el puerto LDAP estándar, TCP 389:

```bash
sudo ss -lntp | grep 389
```

![Figura 15 — Puerto LDAP 389](capturas/Captura%20de%20pantalla%202026-10-09%20185217.png)

*Figura 15. Comprobación del proceso `slapd` en el puerto `389`.*

### C.3. Verificación de la base DN

Se consultó el contenido de la base LDAP:

```bash
ldapsearch -x -LLL -b dc=planetafp,dc=local
```

![Figura 16 — Base DN de OpenLDAP](capturas/Captura%20de%20pantalla%202026-10-09%20185249.png)

*Figura 16. Consulta de la base del directorio `dc=planetafp,dc=local`.*

### C.4. Esquemas LDAP disponibles

Se verificaron los esquemas disponibles en el sistema:

```bash
ls -l /etc/ldap/schema/
```

![Figura 17 — Esquemas LDAP](capturas/Captura%20de%20pantalla%202026-10-09%20185320.png)

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

![Figura 18 — Fichero LDIF inicial](capturas/Captura%20de%20pantalla%202026-10-09%20185306.png)

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

![Figura 19 — Inserción con ldapadd](capturas/Captura%20de%20pantalla%202026-10-09%20185530.png)

*Figura 19. Alta de la OU y el usuario mediante `ldapadd`.*

### D.3. Búsqueda por identificador de usuario

Se realizó una búsqueda delimitando la base, el ámbito y el filtro por UID:

```bash
ldapsearch -x -LLL \
  -b "ou=personas,dc=planetafp,dc=local" \
  -s sub \
  "(uid=raul)"
```

![Figura 20 — Búsqueda LDAP por UID](capturas/Captura%20de%20pantalla%202026-10-09%20185858.png)

*Figura 20. Consulta LDAP del usuario cuyo atributo `uid` es `raul`.*

### D.4. Búsqueda por clase de objeto

También se realizó una búsqueda sobre toda la base del directorio para recuperar las entradas de tipo `inetOrgPerson`:

```bash
ldapsearch -x -LLL \
  -b "dc=planetafp,dc=local" \
  -s sub \
  "(objectClass=inetOrgPerson)"
```

![Figura 21 — Búsqueda por objectClass](capturas/Captura%20de%20pantalla%202026-10-09%20185949.png)

*Figura 21. Consulta LDAP filtrada por la clase de objeto `inetOrgPerson`.*

### D.5. Verificación de identidad del administrador

Se comprobó el bind autenticado del administrador LDAP:

```bash
ldapwhoami -x -D "cn=admin,dc=planetafp,dc=local" -W
```

![Figura 22 — Bind del administrador](capturas/Captura%20de%20pantalla%202026-10-09%20190041.png)

*Figura 22. Verificación de la identidad autenticada del administrador LDAP.*

---

## E. Control de Errores y Troubleshooting

### E.1. Error: entrada duplicada `Already exists (68)`

Durante una prueba se intentó volver a crear una entrada que ya existía en el directorio:

```bash
ldapadd -x -D "cn=admin,dc=planetafp,dc=local" -W -f error-sintaxis.ldif
```

La entrada que se intentaba insertar era:

```text
uid=prueba,ou=personas,dc=planetafp,dc=local
```

La operación devolvió:

```text
ldap_add: Already exists (68)
```

![Figura 23 — Error de entrada duplicada](capturas/Captura%20de%20pantalla%202026-10-09%20183923.png)

*Figura 23. El servidor LDAP rechaza la creación de un DN que ya existe.*

**Causa:** el DN `uid=prueba,ou=personas,dc=planetafp,dc=local` ya estaba registrado. LDAP no permite crear dos entradas con el mismo DN.

**Diagnóstico:**

```bash
ldapsearch -x -LLL \
  -b "ou=personas,dc=planetafp,dc=local" \
  "(uid=prueba)"
```

**Solución:** si se desea modificar una entrada existente debe utilizarse `ldapmodify`, no `ldapadd`. Si es necesario eliminarla y volver a crearla:

```bash
ldapdelete -x -D "cn=admin,dc=planetafp,dc=local" -W \
  "uid=prueba,ou=personas,dc=planetafp,dc=local"
```

Después de eliminar la entrada, se puede repetir el comando `ldapadd` con el fichero LDIF correspondiente.

---

### E.2. Error: credenciales incorrectas `Invalid credentials (49)`

Durante una operación de inserción se introdujo una contraseña incorrecta para el administrador LDAP:

```bash
ldapadd -x -D "cn=admin,dc=planetafp,dc=local" -W -f estructura-inicial.ldif
```

El servidor devolvió:

```text
ldap_bind: Invalid credentials (49)
```

![Figura 24 — Error de credenciales LDAP](capturas/Captura%20de%20pantalla%202026-10-09%20183931.png)

*Figura 24. Fallo de autenticación al introducir una contraseña incorrecta durante el bind LDAP.*

**Causa:** la contraseña introducida para el DN `cn=admin,dc=planetafp,dc=local` no era correcta.

**Diagnóstico y solución:** se verificó la identidad del administrador utilizando la contraseña correcta:

```bash
ldapwhoami -x -D "cn=admin,dc=planetafp,dc=local" -W
```

Resultado:

```text
dn:cn=admin,dc=planetafp,dc=local
```

Esto confirmó que el DN del administrador y el servicio LDAP estaban configurados correctamente. La incidencia se resolvió introduciendo la contraseña correcta. La evidencia del bind correcto se muestra en la Figura 22.

---

### E.3. Error: servidor LDAP no accesible

Con el servicio `slapd` detenido, se ejecutó la siguiente consulta:

```bash
ldapsearch -x -LLL -b dc=planetafp,dc=local
```

El resultado fue:

```text
ldap_sasl_bind(SIMPLE): Can't contact LDAP server (-1)
```

![Figura 25 — Error de conexión con LDAP](capturas/Captura%20de%20pantalla%202026-10-09%20184327.png)

*Figura 25. El cliente LDAP no puede contactar con el servidor porque el servicio `slapd` no está disponible.*

**Causa:** el servicio `slapd` se encontraba detenido, por lo que no había ningún proceso escuchando en el puerto TCP 389.

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

La recuperación se demuestra en las Figuras 14, 15 y 16: `slapd` aparece en estado `active (running)`, el puerto 389 queda en escucha y la consulta LDAP devuelve la entrada raíz del directorio.

---

## Conclusión

Se ha desplegado y configurado correctamente un servidor OpenLDAP en Ubuntu Server 24.04.5 LTS dentro de una máquina virtual VirtualBox. El servidor tiene configurada la IP estática `20.0.0.5/24`, resolución local para `ldapserver.planetafp.local` y una base LDAP correspondiente al dominio `planetafp.local`.

La base `dc=planetafp,dc=local` se verificó correctamente y fue poblada mediante un archivo LDIF con la unidad organizativa `ou=personas` y el usuario `uid=raul`. Las operaciones de inserción, búsqueda y autenticación se realizaron satisfactoriamente con `ldapadd`, `ldapsearch` y `ldapwhoami`.

También se documentaron errores de entradas duplicadas, credenciales incorrectas y falta de conectividad con el servidor LDAP, indicando su causa, diagnóstico y solución.
