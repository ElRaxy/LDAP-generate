# 🌳 Guía LDAP — Ubuntu Server (OpenLDAP) + Cliente

## Índice

- [0. Escenario y Herramientas](https://www.google.com/search?q=%230-escenario-y-herramientas)
- [1. Instalación y Configuración Servidor](https://www.google.com/search?q=%231-instalaci%C3%B3n-y-configuraci%C3%B3n-servidor)
- [2. Carga Masiva (Uso de la Herramienta HTML)](https://www.google.com/search?q=%232-carga-masiva-uso-de-la-herramienta-html)
- [3. Configuración del Cliente (Logueo LDAP)](https://www.google.com/search?q=%233-configuraci%C3%B3n-del-cliente-logueo-ldap)
- [4. Comprobación y Troubleshooting](https://www.google.com/search?q=%234-comprobaci%C3%B3n-y-troubleshooting)

---

## 0. Escenario y Herramientas

- **IP Servidor:** `11.0.21.10` | **Dominio:** `amrdaw.local`
- **IP Cliente:** `11.0.21.50`
- **Herramienta LDIF:** [🚀 Generador Pro](https://ldap-generate-wxb3-git-master-alexms-projects-4a3a1365.vercel.app/) (Usa esto para crear `init.ldif` y `mod.ldif`).

---

## 1. Instalación y Configuración Servidor

```bash
sudo apt update && sudo apt install -y slapd ldap-utils
# Reconfigurar para asegurar dominio correcto
sudo dpkg-reconfigure -plow slapd

```

_(Configuración: DNS=`amrdaw.local`, Org=`amrdaw`, Admin Pass=`tu_clave`, Quitar base de datos: **Sí**)_

---

## 2. Carga Masiva (Uso de la Herramienta HTML)

1. **En el Generador:** Configura la ruta (ej. `/tmp/`) y el nombre (`init.ldif`).
2. **Pegar lista:** `usuario, grupo`.
3. **En la Terminal del Servidor:** Copia el bloque `cat <<EOF` para crear el archivo y luego:

```bash
ldapadd -x -D "cn=admin,dc=amrdaw,dc=local" -W -f /tmp/init.ldif

```

---

## 3. Configuración del Cliente (Logueo LDAP)

En la máquina **cliente**, necesitamos que el sistema "mire" hacia LDAP para buscar usuarios.

### 3.1 Instalación de paquetes

```bash
sudo apt update
sudo apt install -y libnss-ldap libpam-ldap ldap-utils

```

**Durante el asistente de instalación:**

- LDAP server URI: `ldap://11.0.21.10`
- Distinguished name (Search base): `dc=amrdaw,dc=local`
- LDAP version: `3`
- Make local root Database admin: **Yes**
- Does LDAP database require login? **No**

### 3.2 Configurar NSS (Name Service Switch)

Dile al sistema que busque usuarios en LDAP:

```bash
sudo nano /etc/nsswitch.conf

```

Modifica estas tres líneas añadiendo `ldap`:

```text
passwd:         files systemd ldap
group:          files systemd ldap
shadow:         files ldap

```

### 3.3 Crear carpetas HOME automáticamente

Para que al loguearte con un usuario de LDAP se cree su `/home/usuario`:

```bash
sudo nano /etc/pam.d/common-session

```

Añade esta línea al final del archivo:

```text
session required pam_mkhomedir.so skel=/etc/skel umask=077

```

### 3.4 Reiniciar servicios

```bash
sudo systemctl restart nscd
# Si no tienes nscd, no pasa nada, el cambio en PAM/NSS es instantáneo.

```

---

## 4. Comprobación y Troubleshooting

### Comprobar si el cliente "ve" a los usuarios LDAP:

```bash
# Debería devolver los datos del usuario creado con tu HTML
getent passwd pepe
id pepe

```

### Probar logueo local:

```bash
su - pepe

```

### Errores en el logueo (Troubleshooting):

| Síntoma                         | Causa                                       | Solución                                               |
| ------------------------------- | ------------------------------------------- | ------------------------------------------------------ |
| `getent` no devuelve nada       | Error en URI o Base DN en el cliente        | `sudo dpkg-reconfigure ldap-auth-config`               |
| Pide clave pero falla           | Password en el LDIF no es SHA o es distinta | Regenera el LDIF con tu herramienta asegurando el Hash |
| Loguea pero dice "No directory" | Falta la línea en `common-session`          | Añadir `pam_mkhomedir.so`                              |
| El servidor no responde         | Firewall activo                             | `sudo ufw allow 389/tcp`                               |

---
