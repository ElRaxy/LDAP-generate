# GUIA COMPLETA CONFIGURACION DNS Y LDAP

## 1. PREPARACION DEL SERVIDOR (AMRDAW.LOCAL)

**[SERVIDOR]** Ejecutar estas tareas para limpiar el entorno previo y configurar la red.

1. **Cambio de Identidad**
```bash
sudo hostnamectl set-hostname ldap-server.amrdaw.local

```


2. **Configuracion de Hosts**
`sudo nano /etc/hosts`
```text
127.0.0.1 localhost
127.0.1.1 ldap-server.amrdaw.local ldap-server
192.168.200.50 ldap-server.amrdaw.local ldap-server

```


3. **Configuracion de DNS (Bind9)**
`sudo nano /etc/bind/named.conf.options`
* Forwarders: Introducir la IP que devuelva el comando `ip r` o 10.2.1.254 si no va(10.0.2.2) (el gateway).
* Allow-query: Permitir la red local: `localhost; 192.168.200.0/24;`.


1. **Adaptacion de Zonas DNS**
* Renombrar archivo: `sudo mv /etc/bind/zonas/db.emsdaw.local /etc/bind/zonas/db.amrdaw.local`
* Editar contenido: `sudo nano /etc/bind/zonas/db.amrdaw.local` (Reemplazar todas las menciones de emsdaw por amrdaw).
* Vincular zona: `sudo nano /etc/bind/named.conf.local` (Cambiar el nombre de zona a `amrdaw.local` y la ruta del archivo).
* Reiniciar: `sudo systemctl restart bind9`



---

## 2. INSTALACION Y CARGA DE LDAP

**[SERVIDOR]**

1. **Instalacion**
```bash
sudo apt update && sudo apt install -y slapd ldap-utils

```


2. **Reconfiguracion (Asistente)**
`sudo dpkg-reconfigure -plow slapd`
* Omitir configuracion: No
* Dominio: `amrdaw.local`
* Organizacion: `amrdaw`
* Contraseña: `tu_contraseña`
* Mover base de datos antigua: **SI** (esto limpia el rastro del examen anterior).


3. **Carga de Datos (LDIF)**
* Generar el contenido con la [Heramienta de Generacion de LDIF](https://ldap-generate-wxb3.vercel.app/).
* Crear archivo: `cat <<EOF > /tmp/init.ldif` (pegar contenido) `EOF`.
* Inyectar: `ldapadd -x -D "cn=admin,dc=amrdaw,dc=local" -W -f /tmp/init.ldif`



---

## 3. CONFIGURACION DEL CLIENTE

**[CLIENTE]**

1. **Instalacion de Modulos de Autenticacion**
```bash
sudo apt install -y libnss-ldap libpam-ldap ldap-utils

```


* URI: `ldap://192.168.200.50`
* Base DN: `dc=amrdaw,dc=local`
* LDAP Version: 3
* Make local root Database admin: Yes
* Does LDAP database require login: No


2. **Configuracion NSS (Name Service Switch)**
`sudo gedit /etc/nsswitch.conf`
Añadir `ldap` al final de estas lineas:
```text
passwd:         files systemd ldap
group:          files systemd ldap
shadow:         files ldap

```


3. **Configuracion PAM (Creacion Automatica de Home)**
`sudo gedit /etc/pam.d/common-session`
Añadir al final de todo el archivo:
```text
session optional pam_mkhomedir.so skel=/etc/skel umask=077

```



---

## 4. MODIFICACIONES EN CALIENTE

**[SERVIDOR]**

Si necesitas borrar un usuario, añadirlo a un grupo extra o cambiar su loginShell:

1. Generar el LDIF en la pestaña Modificaciones de la herramienta Vercel.
2. Crear el archivo: `cat <<EOF > /tmp/mod.ldif` (pegar contenido) `EOF`.
3. Aplicar cambios:
```bash
ldapmodify -x -D "cn=admin,dc=amrdaw,dc=local" -W -f /tmp/mod.ldif

```



---

## 5. COMPROBACIONES FINALES

| Comando | Ubicacion | Resultado Esperado |
| --- | --- | --- |
| `nslookup google.es` | **[SERVIDOR]** | Debe resolver (Forwarders correctos). |
| `getent passwd pepe` | **[CLIENTE]** | Debe mostrar la linea del usuario LDAP. |
| `su - pepe` | **[CLIENTE]** | Debe iniciar sesion y crear el directorio /home/pepe. |
| `ldapsearch -x -b "dc=amrdaw,dc=local"` | **[SERVIDOR]** | Debe listar todo el arbol del directorio. |


