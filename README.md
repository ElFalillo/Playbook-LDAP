# Playbook-LDAP
# Playbook de Despliegue Técnico  
## Instalación y configuración de OpenLDAP

**Alumno/a:** Raul Ortiz  
**Módulo / Práctica:** Servicio de directorio OpenLDAP  
**Fecha:** 09/10/2026  
**Entorno:** Máquina virtual Ubuntu Server en VirtualBox  

---

## A. Ficha Técnica y Prerrequisitos del Entorno

### A.1. Resumen del entorno

| Elemento | Configuración |
|---|---|
| Hipervisor | Oracle VirtualBox |
| Sistema operativo | Ubuntu Server |
| Tipo de red | Red NAT sin DHCP |
| Dirección IP estática | `20.0.0.5/24` |
| Interfaz de red | `enp0s3` |
| FQDN | `ldapserver.planetafp.local` |
| Alias | `ldapserver` |
| Dominio del directorio | `planetafp.local` |
| Base DN | `dc=planetafp,dc=local` |
| Organización | `planetafp` |
| Administrador LDAP | `cn=admin,dc=planetafp,dc=local` |
| Paquetes instalados | `slapd`, `ldap-utils`, `libldap-common` |
| Puerto LDAP | `389/tcp` |

### A.2. Descripción del servicio de directorio

OpenLDAP es una implementación open source del protocolo LDAP (*Lightweight Directory Access Protocol*), utilizado para almacenar y consultar información jerárquica como usuarios, grupos, unidades organizativas y credenciales.

Los componentes principales son:

- **slapd**: demonio servidor LDAP. Escucha peticiones en el puerto 389 y gestiona la base de datos del directorio.
- **ldap-utils**: conjunto de herramientas cliente, como `ldapadd`, `ldapsearch`, `ldapmodify` y `ldapdelete`.
- **libldap**: librerías necesarias para que las aplicaciones y utilidades se comuniquen mediante el protocolo LDAP.
- **Esquemas**: definen los tipos de objetos y atributos permitidos en el directorio, como `inetOrgPerson`, `organizationalUnit` o `posixAccount`.

### A.3. Capturas de prerrequisitos

**Captura 1 — Versión del sistema operativo**

```bash
lsb_release -a
```

`[ INSERTAR CAPTURA ]`

**Captura 2 — Paquetes instalados**

```bash
dpkg -l | grep -E "slapd|ldap-utils|libldap"
```

`[ INSERTAR CAPTURA ]`

---

## B. Procedimiento de Despliegue Paso a Paso

### Paso 1 — Configuración de red estática con Netplan

En Ubuntu Server, la configuración de red se define mediante Netplan en archivos YAML ubicados en `/etc/netplan/`.

1. Identificar la interfaz de red:

```bash
ip a
```

`[ INSERTAR CAPTURA: interfaz enp0s3 ]`

2. Crear el archivo de configuración:

```bash
sudo nano /etc/netplan/01-ldap.yaml
```

3. Añadir el siguiente contenido:

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

4. Aplicar la configuración:

```bash
sudo netplan apply
```

5. Verificar la configuración:

```bash
ip a
ip route
```

**Captura 3 — Archivo Netplan**

```bash
cat /etc/netplan/01-ldap.yaml
```

`[ INSERTAR CAPTURA ]`

**Captura 4 — Aplicación de Netplan**

```bash
sudo netplan apply
```

`[ INSERTAR CAPTURA ]`

**Captura 5 — IP estática asignada**

```bash
ip a
```

Debe verse:

```text
inet 20.0.0.5/24
```

`[ INSERTAR CAPTURA ]`

**Captura 6 — Tabla de rutas**

```bash
ip route
```

`[ INSERTAR CAPTURA ]`

---

### Paso 2 — Resolución de nombres local

Se añade el FQDN y el alias al archivo `/etc/hosts` para que el sistema pueda resolver el nombre del servidor sin depender de un servidor DNS externo.

1. Editar el archivo:

```bash
sudo nano /etc/hosts
```

2. Añadir la siguiente línea:

```text
20.0.0.5    ldapserver.planetafp.local ldapserver
```

3. Guardar y comprobar:

```bash
ping -c 3 ldapserver.planetafp.local
ping -c 3 ldapserver
```

**Justificación:** esta configuración permite que el servidor se identifique correctamente mediante su FQDN, evita errores de resolución durante la configuración de `slapd` y permite utilizar el nombre `ldapserver` en las conexiones LDAP.

**Captura 7 — Contenido de `/etc/hosts`**

```bash
cat /etc/hosts
```

`[ INSERTAR CAPTURA ]`

**Captura 8 — Resolución del FQDN**

```bash
ping -c 3 ldapserver.planetafp.local
```

`[ INSERTAR CAPTURA ]`

**Captura 9 — Resolución del alias**

```bash
ping -c 3 ldapserver
```

`[ INSERTAR CAPTURA ]`

---

### Paso 3 — Instalación del software

1. Actualizar los repositorios:

```bash
sudo apt update
```

2. Instalar OpenLDAP y las utilidades cliente:

```bash
sudo apt install -y slapd ldap-utils libldap-common
```

**Captura 10 — Actualización de repositorios**

```bash
sudo apt update
```

`[ INSERTAR CAPTURA ]`

**Captura 11 — Instalación de paquetes**

```bash
sudo apt install -y slapd ldap-utils libldap-common
```

`[ INSERTAR CAPTURA ]`

**Captura 12 — Verificación de paquetes**

```bash
dpkg -l | grep -E "slapd|ldap-utils|libldap"
```

`[ INSERTAR CAPTURA ]`

---

### Paso 4 — Reconfiguración del servicio `slapd`

Se ejecuta el asistente de reconfiguración:

```bash
sudo dpkg-reconfigure slapd
```

**Respuestas utilizadas:**

| Pregunta | Respuesta |
|---|---|
| ¿Omitir la configuración del servidor OpenLDAP? | **No** |
| Nombre del dominio DNS | `planetafp.local` |
| Nombre de la organización | `planetafp` |
| Contraseña del administrador | Contraseña segura establecida durante la práctica |
| Confirmación de contraseña | La misma contraseña |
| Motor de base de datos | **MDB** |
| ¿Eliminar la base de datos al purgar `slapd`? | **No** |
| ¿Mover la base de datos antigua? | **Sí** |
| ¿Permitir el protocolo LDAPv2? | **No** |

Como resultado, se crea la base del directorio:

```text
dc=planetafp,dc=local
```

y la cuenta administradora:

```text
cn=admin,dc=planetafp,dc=local
```

**Capturas 13 a 17 — Proceso interactivo**

`[ INSERTAR CAPTURA: Omitir configuración = No ]`  
`[ INSERTAR CAPTURA: Dominio planetafp.local ]`  
`[ INSERTAR CAPTURA: Organización planetafp ]`  
`[ INSERTAR CAPTURA: Contraseña administrador ]`  
`[ INSERTAR CAPTURA: No eliminar base de datos al purgar ]`

---

## C. Verificación y Comprobación

### C.1. Estado del servicio

```bash
sudo systemctl status slapd
```

Resultado esperado:

```text
Active: active (running)
```

**Captura 18 — Estado del servicio**

`[ INSERTAR CAPTURA ]`

### C.2. Comprobación del arranque automático

```bash
sudo systemctl is-enabled slapd
```

Resultado esperado:

```text
enabled
```

**Captura 19 — Servicio habilitado**

`[ INSERTAR CAPTURA ]`

### C.3. Puerto LDAP en escucha

```bash
sudo ss -lntp | grep 389
```

**Captura 20 — Puerto 389**

`[ INSERTAR CAPTURA ]`

### C.4. Consulta de la base del directorio

```bash
ldapsearch -x -LLL -b dc=planetafp,dc=local
```

Resultado esperado:

```text
dn: dc=planetafp,dc=local
objectClass: top
objectClass: dcObject
objectClass: organization
o: planetafp
dc: planetafp

dn: cn=admin,dc=planetafp,dc=local
```

**Captura 21 — Base DN creada**

`[ INSERTAR CAPTURA ]`

### C.5. Esquemas disponibles

```bash
ls -l /etc/ldap/schema/
```

**Captura 22 — Esquemas del sistema**

`[ INSERTAR CAPTURA ]`

### C.6. Esquemas cargados en la configuración

```bash
sudo ldapsearch -Q -LLL -Y EXTERNAL -H ldapi:/// -b cn=config dn
```

**Captura 23 — Esquemas cargados**

`[ INSERTAR CAPTURA ]`

---

## D. Población del Directorio y Operaciones Básicas

### D.1. Creación del fichero LDIF

Se crea el archivo:

```bash
nano estructura-inicial.ldif
```

Contenido:

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
userPassword: {SSHA}ContrasenaSegura
```

**Captura 24 — Fichero LDIF**

```bash
cat estructura-inicial.ldif
```

`[ INSERTAR CAPTURA ]`

### D.2. Inserción con `ldapadd`

```bash
ldapadd -x -D "cn=admin,dc=planetafp,dc=local" -W -f estructura-inicial.ldif
```

Parámetros utilizados:

- `-x`: autenticación simple.
- `-D`: DN del administrador para realizar el bind.
- `-W`: solicita la contraseña de forma interactiva.
- `-f`: indica el fichero LDIF desde el que se cargarán las entradas.

**Captura 25 — Alta de la OU y usuario**

`[ INSERTAR CAPTURA ]`

Salida esperada:

```text
adding new entry "ou=personas,dc=planetafp,dc=local"

adding new entry "uid=raul,ou=personas,dc=planetafp,dc=local"
```

### D.3. Consulta con `ldapsearch`

Búsqueda del usuario por UID:

```bash
ldapsearch -x -LLL \
  -b "ou=personas,dc=planetafp,dc=local" \
  -s sub \
  "(uid=raul)"
```

**Captura 26 — Búsqueda por UID**

`[ INSERTAR CAPTURA ]`

Búsqueda por clase de objeto:

```bash
ldapsearch -x -LLL \
  -b "dc=planetafp,dc=local" \
  -s sub \
  "(objectClass=inetOrgPerson)"
```

**Captura 27 — Búsqueda por objectClass**

`[ INSERTAR CAPTURA ]`

### D.4. Comprobación del bind administrador

```bash
ldapwhoami -x -D "cn=admin,dc=planetafp,dc=local" -W
```

**Captura 28 — Identidad del administrador**

`[ INSERTAR CAPTURA ]`

---

## E. Control de Errores y Troubleshooting

### E.1. Fallo 1: Entrada duplicada — `Already exists (68)`

**Error observado:**

```text
ldap_add: Already exists (68)
```

**Causa:**

El error se produjo al ejecutar nuevamente el comando `ldapadd` con el fichero `estructura-inicial.ldif`. Las entradas `ou=personas,dc=planetafp,dc=local` y `uid=raul,ou=personas,dc=planetafp,dc=local` ya existían en el directorio, por lo que OpenLDAP impidió crear un DN duplicado.

**Comando que provocó el error:**

```bash
ldapadd -x -D "cn=admin,dc=planetafp,dc=local" -W -f estructura-inicial.ldif
```

**Diagnóstico:**

```bash
ldapsearch -x -LLL -b dc=planetafp,dc=local
```

La consulta confirmó que la entrada ya existía.

**Solución aplicada:**

Se eliminó la entrada duplicada antes de volver a crearla:

```bash
ldapdelete -x -D "cn=admin,dc=planetafp,dc=local" -W \
  "uid=raul,ou=personas,dc=planetafp,dc=local"
```

Posteriormente se volvió a insertar:

```bash
ldapadd -x -D "cn=admin,dc=planetafp,dc=local" -W -f estructura-inicial.ldif
```

**Comprobación final:**

```bash
ldapsearch -x -LLL -b ou=personas,dc=planetafp,dc=local "(uid=raul)"
```

**Captura 29 — Error `Already exists (68)`**

`[ INSERTAR CAPTURA ]`

**Captura 30 — Comprobación de que la entrada existía**

`[ INSERTAR CAPTURA ]`

**Captura 31 — Solución aplicada**

`[ INSERTAR CAPTURA ]`

**Captura 32 — Verificación final**

`[ INSERTAR CAPTURA ]`

---

### E.2. Fallo 2: Servicio `slapd` detenido — `Can't contact LDAP server (-1)`

**Error observado:**

```text
ldap_sasl_bind(SIMPLE): Can't contact LDAP server (-1)
```

**Causa:**

El demonio `slapd` fue detenido manualmente con `systemctl stop slapd`. Al no estar activo, el cliente LDAP no pudo establecer conexión con el servidor en el puerto 389.

**Comandos que provocaron el error:**

```bash
sudo systemctl stop slapd
ldapsearch -x -LLL -b dc=planetafp,dc=local
```

**Diagnóstico:**

```bash
sudo systemctl status slapd
```

Salida obtenida:

```text
Active: inactive (dead)
```

También se comprobó que el puerto 389 no estaba en escucha:

```bash
sudo ss -lntp | grep 389
```

**Solución aplicada:**

```bash
sudo systemctl start slapd
```

**Comprobación de la solución:**

```bash
sudo systemctl status slapd
```

Salida esperada:

```text
Active: active (running)
```

Verificación del acceso al directorio:

```bash
ldapsearch -x -LLL -b dc=planetafp,dc=local
```

**Captura 33 — Detención del servicio**

`[ INSERTAR CAPTURA ]`

**Captura 34 — Servicio parado**

`[ INSERTAR CAPTURA ]`

**Captura 35 — Error de conexión LDAP**

`[ INSERTAR CAPTURA ]`

**Captura 36 — Arranque del servicio**

`[ INSERTAR CAPTURA ]`

**Captura 37 — Servicio activo**

`[ INSERTAR CAPTURA ]`

**Captura 38 — Búsqueda LDAP correcta**

`[ INSERTAR CAPTURA ]`

---

## Conclusión técnica

Se ha desplegado correctamente un servidor OpenLDAP en Ubuntu Server sobre VirtualBox, con dirección IP estática `20.0.0.5/24`, resolución local del FQDN `ldapserver.planetafp.local` e instalación de los paquetes `slapd` y `ldap-utils`.

El directorio `dc=planetafp,dc=local` ha sido configurado, verificado y poblado con una unidad organizativa y un usuario de prueba. Además, se han documentado dos incidencias reales: una entrada duplicada y la detención del servicio `slapd`, incluyendo su diagnóstico y solución.
