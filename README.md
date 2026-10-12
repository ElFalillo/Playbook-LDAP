<details>
<summary><h2><b> 👩‍💻LDAP/SERVER </b></h2></summary>

# Playbook de despliegue técnico
## Instalación y configuración de OpenLDAP

**Autores:** Falillo y JAM  
**Entorno:** Ubuntu Server 24.04.5 LTS en Oracle VirtualBox  
**Servicio:** OpenLDAP  
**Fecha de realización:** 09/10/2026

---

## 1. Entorno y requisitos

| Elemento | Configuración |
|---|---|
| Hipervisor | Oracle VirtualBox |
| Sistema operativo | Ubuntu Server 24.04.5 LTS |
| Nombre del equipo | `UbuntuServer` |
| Tipo de red | NAT sin DHCP |
| Interfaz de red | `enp0s3` |
| Dirección IP | `20.0.0.5/24` |
| Puerta de enlace | `20.0.0.1` |
| DNS | `20.0.0.1` y `8.8.8.8` |
| FQDN | `ldapserver.planetafp.local` |
| Alias | `ldapserver` |
| Dominio LDAP | `planetafp.local` |
| Base DN | `dc=planetafp,dc=local` |
| Administrador LDAP | `cn=admin,dc=planetafp,dc=local` |
| Puerto LDAP | TCP 389 |

### 1.1. Comprobar el sistema operativo

Antes de iniciar el despliegue se comprobó la versión de Ubuntu Server:

```bash
lsb_release -a
```

![Información de Ubuntu Server](capturas/Captura%20de%20pantalla%202026-10-09%20183525.png)

*La captura corresponde a la comprobación de Ubuntu Server 24.04.5 LTS.*

### 1.2. Comprobar los paquetes LDAP instalados

Se consultaron los paquetes relacionados con OpenLDAP:

```bash
dpkg -l | grep -E "slapd|ldap-utils|libldap"
```

![Paquetes LDAP instalados](capturas/Captura%20de%20pantalla%202026-10-09%20183646.png)

*La captura debe mostrar los paquetes LDAP instalados, como `slapd`, `ldap-utils` y las bibliotecas `libldap`.*

---

## 2. Configuración de red

### 2.1. Asignar una IP estática con Netplan

La máquina virtual utiliza NAT sin DHCP. Se configuró una dirección estática para que el servidor mantenga la IP `20.0.0.5/24`.

Archivo configurado:

```text
/etc/netplan/50-cloud-init.yaml
```

Se aplicaron los cambios con:

```bash
sudo netplan apply
```

### 2.2. Verificar la dirección IP

```bash
ip a
```

![Verificación de la interfaz y dirección IP](capturas/Captura%20de%20pantalla%202026-10-09%20183539.png)

*La captura se utiliza para verificar la dirección asignada a `enp0s3`.*

### 2.3. Verificar la tabla de rutas

```bash
ip route
```

![Verificación de rutas](capturas/Captura%20de%20pantalla%202026-10-09%20183551.png)

*La captura se utiliza para comprobar la ruta por defecto a través de `20.0.0.1`.*

---

## 3. Resolución local de nombres

Como no se configuró un DNS interno para `planetafp.local`, se añadió una entrada local en `/etc/hosts`:

```text
20.0.0.5    ldapserver.planetafp.local ldapserver
```

Esta entrada permite utilizar el FQDN y el alias corto para localizar el servidor.

### 3.1. Revisar `/etc/hosts`

```bash
cat /etc/hosts
```

![Entrada local de /etc/hosts](capturas/Captura%20de%20pantalla%202026-10-09%20183803.png)

*La captura debe mostrar la asociación de `20.0.0.5` con `ldapserver.planetafp.local` y `ldapserver`.*

### 3.2. Probar la resolución del FQDN y del alias

```bash
ping ldapserver.planetafp.local
ping ldapserver
```

![Pruebas de resolución de nombres](capturas/Captura%20de%20pantalla%202026-10-09%20183749.png)

*La captura se utiliza para comprobar la resolución del FQDN y del alias.*

---

## 4. Instalación de OpenLDAP

### 4.1. Actualizar los repositorios

```bash
sudo apt update
```

![Actualización de repositorios](capturas/Captura%20de%20pantalla%202026-10-09%20183821.png)

*La captura corresponde a la actualización de los índices de paquetes de Ubuntu.*

### 4.2. Instalar el servicio y las herramientas

```bash
sudo apt install -y slapd ldap-utils libldap-common
```

![Instalación de OpenLDAP](capturas/Captura%20de%20pantalla%202026-10-09%20183857.png)

*La captura corresponde a la instalación de `slapd`, `ldap-utils` y `libldap-common`.*

---

## 5. Reconfiguración del servicio `slapd`

Se ejecutó el asistente de configuración:

```bash
sudo dpkg-reconfigure slapd
```

Los parámetros elegidos durante la práctica fueron:

| Pregunta del asistente | Valor seleccionado |
|---|---|
| Omitir la configuración de OpenLDAP | No |
| Nombre DNS del dominio | `planetafp.local` |
| Organización | `planetafp` |
| Motor de base de datos | MDB |
| Eliminar la base de datos al purgar `slapd` | No |
| Mover la base de datos anterior | Sí |
| Permitir LDAPv2 | No |
| Contraseña administrativa | Configurada durante la práctica |

El dominio genera esta base DN:

```text
dc=planetafp,dc=local
```

El DN administrativo es:

```text
cn=admin,dc=planetafp,dc=local
```

### 5.1. Organización

![Configuración de la organización](capturas/Captura%20de%20pantalla%202026-10-09%20183931.png)

*Captura asociada a la selección de la organización `planetafp`.*

### 5.2. Dominio LDAP

![Configuración del dominio LDAP](capturas/Captura%20de%20pantalla%202026-10-09%20183923.png)

*Captura asociada al dominio `planetafp.local`.*

---

## 6. Verificación del servicio LDAP

### 6.1. Estado del servicio

```bash
sudo systemctl status slapd
```

![Estado del servicio slapd](capturas/Captura%20de%20pantalla%202026-10-09%20184121.png)

*Usar esta captura para documentar el estado que devuelve `systemctl status slapd`.*

### 6.2. Puerto LDAP en escucha

```bash
sudo ss -lntp | grep 389
```

![Comprobación del puerto LDAP](capturas/Captura%20de%20pantalla%202026-10-09%20184327.png)

*Usar esta captura para documentar la comprobación de TCP 389.*

### 6.3. Consultar la base DN

```bash
ldapsearch -x -LLL -b "dc=planetafp,dc=local"
```

![Consulta de la base DN](capturas/Captura%20de%20pantalla%202026-10-09%20184401.png)

*Usar esta captura para documentar la consulta de la entrada raíz del directorio.*

### 6.4. Consultar los esquemas disponibles

```bash
ls -l /etc/ldap/schema/
```

![Esquemas LDAP disponibles](capturas/Captura%20de%20pantalla%202026-10-09%20184421.png)

*La captura debe mostrar los esquemas disponibles, entre ellos `core`, `cosine`, `inetorgperson` y `nis`.*

---

## 7. Población del directorio LDAP

### 7.1. Crear la estructura inicial en LDIF

El archivo `estructura-inicial.ldif` crea la OU `ou=personas` y el usuario `uid=raul`.

![Archivo LDIF inicial](capturas/Captura%20de%20pantalla%202026-10-09%20185217.png)

> La contraseña del LDIF es un valor de prueba. Para un entorno real, se debe generar un hash con `slappasswd` y no guardar contraseñas en texto plano.

*La captura corresponde a la edición o revisión del archivo LDIF inicial.*

### 7.2. Añadir las entradas al directorio

```bash
ldapadd -x -D "cn=admin,dc=planetafp,dc=local" -W -f estructura-inicial.ldif
```

| Opción | Función |
|---|---|
| `-x` | Utiliza autenticación simple |
| `-D` | Indica el DN usado para el bind |
| `-W` | Solicita la contraseña de forma interactiva |
| `-f` | Lee las entradas desde el archivo LDIF |

![Inserción de entradas LDAP](capturas/Captura%20de%20pantalla%202026-10-09%20185249.png)

*La captura corresponde a la ejecución de `ldapadd` para cargar la OU y el usuario.*

### 7.3. Buscar el usuario por UID

```bash
ldapsearch -x -LLL \
  -b "ou=personas,dc=planetafp,dc=local" \
  -s sub \
  "(uid=raul)"
```

![Búsqueda LDAP por UID](capturas/Captura%20de%20pantalla%202026-10-09%20185306.png)

*La captura corresponde a la búsqueda del usuario `raul` por su atributo `uid`.*

### 7.4. Comprobar la identidad administrativa

```bash
ldapwhoami -x -D "cn=admin,dc=planetafp,dc=local" -W
```

![Identidad autenticada del administrador](capturas/Captura%20de%20pantalla%202026-10-09%20185530.png)

*La captura corresponde a la comprobación del bind administrativo con `ldapwhoami`.*

---

## 8. Incidencias y resolución

En esta sección se documentan dos errores observados durante la práctica. Añade debajo de cada incidencia la captura correspondiente cuando esté disponible.

### 8.1. credenciales inválidas en el bind LDAP
Al ejecutar ldapadd, el cliente intentó autenticarse en el directorio usando el DN cn=admin,dc=planetafp,dc=local. El servidor respondió ldap_bind: Invalid credentials (49),por lo que no se autenticó la cuenta y no llegó a procesarse la inserción del fichero LDIF.

![Identidad autenticada del administrador](capturas/Captura%20de%20pantalla%202026-10-09%20185949.png)

La incidencia se resolvió utilizando el DN administrativo correcto y la contraseña configurada para el directorio. Se verificó primero el bind con ldapwhoami y, tras autenticarse correctamente, se volvió a ejecutar ldapadd para cargar las entradas del fichero LDIF.

### 8.2. Entrada LDAP ya existente
El servidor rechazó la operación porque la entrada uid=prueba,ou=personas,dc=planetafp,dc=local ya existía.

![Identidad autenticada del administrador](capturas/Captura%20de%20pantalla%202026-10-09%20190041.png)

Usando ldapsearch -x -LLL comprobamos que la entrada uid=prueba ya existía en el directorio.

## Conclusión

Se instaló OpenLDAP en Ubuntu Server 24.04.5 LTS, se configuró el dominio `planetafp.local` y se creó la base `dc=planetafp,dc=local`.
Se añadieron una unidad organizativa y un usuario de prueba, y se realizaron verificaciones del servicio, consultas LDAP y pruebas de autenticación.
En el final se documentaron incidencias durante el proceso.

