# Lab 01 · Ubuntu Server: usuarios, permisos, carpeta compartida y SSH con claves

Configuración de un servidor Ubuntu Server en un entorno virtualizado: gestión de usuarios y grupos, directorios compartidos por grupo con permisos `2770` (setgid) y acceso SSH únicamente con clave pública, con el login de `root` y las contraseñas deshabilitados.

| | |
|---|---|
| **Bloque** | 01 · Linux básico |
| **Dificultad** | Fácil |
| **Sistema** | Ubuntu Server 26.04.1 LTS |
| **Entorno** | Oracle VirtualBox (red interna + NAT) |
| **Realizado** | 7–9 de noviembre de 2026 |
| **Estado** | Completado |

## Índice

1. [Objetivo](#objetivo)
2. [Habilidades demostradas](#habilidades-demostradas)
3. [Requisitos y entorno](#requisitos-y-entorno)
4. [Pasos](#pasos)
5. [Comprobación](#comprobación)
6. [Buenas prácticas aplicadas](#buenas-prácticas-aplicadas)
7. [Problemas encontrados](#problemas-encontrados)
8. [Conclusiones](#conclusiones)

## Objetivo

- Dejar un servidor Linux actualizado, con red configurada y una instantánea inicial a la que poder volver.
- Crear un usuario administrador con `sudo` para no trabajar como `root`, y dos usuarios de prueba asignados a los grupos `dev` y `soporte`.
- Crear un directorio por grupo en el que los archivos nuevos hereden el grupo (`setgid`) y al que solo pueda acceder su grupo.
- Acceder por SSH solo con clave pública, sin permitir el login de `root` ni la autenticación por contraseña.

## Habilidades demostradas

- Gestión de usuarios y grupos (`adduser`, `groupadd`, `usermod`, `id`, `getent`)
- Permisos especiales (`setgid`) con `chown` y `chmod`
- Hardening de SSH (`sshd_config`, `PermitRootLogin`, `PasswordAuthentication`)
- Auditoría básica y verificación de cada cambio
- Uso de instantáneas y copias de seguridad antes de modificar configuración crítica

## Requisitos y entorno

| Elemento | Valor |
|---|---|
| Sistema operativo | Ubuntu Server 26.04.1 LTS (instalado desde la ISO oficial) |
| Virtualizador | Oracle VirtualBox |
| Recursos de la VM | 4 GB de RAM y 2 procesadores |
| Red | Adaptador 1: red interna · Adaptador 2: NAT |
| Cliente SSH | Windows (CMD con OpenSSH) |
| Paquetes | `openssh-server` |

<!-- TODO: en tu Word aparecían también ufw, acl y fail2ban como paquetes instalados, pero ningún paso del laboratorio los usa. Si los instalaste y los usaste, añádelos aquí y documenta el paso; si no, déjalos fuera. "sudo apt update" no es un paquete, lo he quitado de la lista. -->

### Datos del laboratorio

| Elemento | Valor |
|---|---|
| Nombre de la VM | `lab-01-ubuntu-usuarios-ssh` |
| Hostname | `srv-lab01` (el instalador se configuró con `srv-ubuntu01` y se cambió después con `hostnamectl`) |
| Usuario del instalador | `sysadmin` |
| Administrador | `admin01` (grupo `sudo`) |
| Usuarios de prueba | `user-01-dev` · `user-01-soporte` |
| Grupos | `dev` · `soporte` |
| Acceso SSH desde el anfitrión | `127.0.0.1:2222` (reenvío de puertos de la NAT hacia el puerto 22 de la VM) |

**Rutas utilizadas**

| Ruta | Uso |
|---|---|
| `/etc/netplan/00-installer-config.yaml` | Configuración de red |
| `/srv/proyectos` | Directorio compartido del grupo `dev` |
| `/srv/soporte` | Directorio compartido del grupo `soporte` |
| `~/.ssh/authorized_keys` | Claves públicas autorizadas de cada usuario |
| `/etc/ssh/sshd_config` | Configuración del servidor SSH |
| `/etc/ssh/sshd_config.bak` | Copia de seguridad de la configuración original |

## Pasos

### Bloque 1: Instalación y red

Esta fase prepara la máquina virtual y configura sus adaptadores de red.

#### 1.1 Instalación de Ubuntu Server

Se crea una máquina virtual llamada `lab-01-ubuntu-usuarios-ssh` con 4 GB de RAM y 2 procesadores.

![Máquina virtual lab-01-ubuntu-usuarios-ssh en VirtualBox](recursos/imagenes/01-ubuntu-usuarios-ssh/nombre-maquina.png)

![Configuración del sistema de la VM: 4096 MB de memoria y 2 procesadores](recursos/imagenes/01-ubuntu-usuarios-ssh/esp-ram-proc.png)

Se inicia la máquina y se sigue el instalador de Ubuntu Server 26.04 LTS desde la ISO oficial. Solo se indican las opciones relevantes para el laboratorio; el resto se dejó por defecto.

| Opción del instalador | Valor |
|---|---|
| Idioma | Inglés |
| Teclado | Español / Español |
| Base de instalación | Ubuntu Server |

En la pantalla de perfil se define el nombre del servidor (`srv-ubuntu01`) y el usuario inicial (`sysadmin`).

![Pantalla Profile configuration del instalador](recursos/imagenes/01-ubuntu-usuarios-ssh/config-perfil.png)

#### 1.2 Configuración de red

El adaptador 1 se usa como red interna y el adaptador 2 como NAT. Dentro de la VM se corresponden con `enp0s3` y `enp0s8`:

- `enp0s3` (red interna): sin DHCP y con dirección estática en la red `192.168.10.0/24`. VirtualBox no ofrece DHCP en las redes internas.
- `enp0s8` (NAT): configurada por DHCP; es la que da salida a Internet para `apt`.

Para cambiar las IP se edita el archivo de Netplan y se aplican los cambios:

```bash
sudo nano /etc/netplan/00-installer-config.yaml
sudo netplan apply
```

![Archivo netplan con enp0s3 con IP estática y enp0s8 por DHCP](recursos/imagenes/01-ubuntu-usuarios-ssh/config-red.png)

Como una red interna no es accesible desde el PC anfitrión, el acceso SSH se hace mediante una regla de reenvío de puertos en el adaptador NAT (`127.0.0.1:2222` → puerto `22` de la VM). Por eso los comandos SSH de este laboratorio usan `-p 2222` y `127.0.0.1`.

<!-- TODO (opcional): añadir una captura de la regla de reenvío de puertos de VirtualBox (Configuración → Red → Adaptador 2 → Avanzadas → Reenvío de puertos). Esta línea la he deducido de los comandos que usas (-p 2222 ... 127.0.0.1). -->

### Bloque 2: Actualización

Este bloque deja el sistema actualizado, asigna el nombre del equipo y la zona horaria, y crea la instantánea inicial.

#### 2.1 Actualización y asignación

Se actualiza el equipo:

```bash
sudo apt update && sudo apt upgrade -y
```

Se define el nombre del equipo y la zona horaria:

```bash
sudo hostnamectl set-hostname srv-lab01
sudo timedatectl set-timezone Europe/Madrid
```

![Comandos hostnamectl y timedatectl](recursos/imagenes/01-ubuntu-usuarios-ssh/equipo-zona.png)

Se crea la instantánea inicial, a la que se puede volver si algo sale mal en los bloques siguientes.

![Snapshot Inicial en VirtualBox](recursos/imagenes/01-ubuntu-usuarios-ssh/snapshot-inicial.png)

### Bloque 3: Usuarios y grupos

Este bloque crea los usuarios y grupos y añade cada usuario a su grupo.

#### 3.1 Creación de usuarios y asignación de grupos

Se crea un usuario administrador con permisos de `sudo` para no tener que trabajar como `root`:

```bash
sudo adduser admin01
sudo usermod -aG sudo admin01
```

![Creación del usuario admin01 y alta en el grupo sudo](recursos/imagenes/01-ubuntu-usuarios-ssh/user-admin.png)

Se crean los grupos `dev` y `soporte` y se comprueban con `getent`:

```bash
sudo groupadd dev && sudo groupadd soporte
getent group dev && getent group soporte
```

![Creación y comprobación de los grupos dev y soporte](recursos/imagenes/01-ubuntu-usuarios-ssh/dev-soporte-groups.png)

Se crean dos usuarios de prueba:

```bash
sudo adduser user-01-dev
sudo adduser user-01-soporte
```

![Creación de user-01-dev y user-01-soporte con adduser](recursos/imagenes/01-ubuntu-usuarios-ssh/creacion-users.png)

Cada usuario se añade a su grupo y se comprueba con `id`:

```bash
sudo usermod -aG dev user-01-dev
sudo usermod -aG soporte user-01-soporte
id user-01-dev
id user-01-soporte
```

![Asignación de los usuarios a sus grupos y comprobación con id](recursos/imagenes/01-ubuntu-usuarios-ssh/anadir-users-grupo.png)

### Bloque 4: Permisos

En este bloque se asignan los permisos a los directorios de trabajo de cada grupo. Cada directorio pertenece a `root` y a su grupo, con permisos `2770`.

Directorio del grupo `dev`:

```bash
sudo mkdir /srv/proyectos
sudo chown root:dev /srv/proyectos/
sudo chmod 2770 /srv/proyectos/
```

![Creación de /srv/proyectos con propietario root:dev y permisos 2770](recursos/imagenes/01-ubuntu-usuarios-ssh/permisos-dev.png)

Directorio del grupo `soporte`:

```bash
sudo mkdir /srv/soporte
sudo chown root:soporte /srv/soporte/
sudo chmod 2770 /srv/soporte/
```

![Creación de /srv/soporte con propietario root:soporte y permisos 2770](recursos/imagenes/01-ubuntu-usuarios-ssh/permisos-soporte.png)

**Desglose de los permisos `2770`**

| Dígito | Permiso | Efecto |
|---|---|---|
| 2 | `setgid` | Los archivos y subcarpetas nuevos heredan el grupo de la carpeta, no el grupo principal de quien los crea |
| 7 | `rwx` (propietario) | El propietario puede leer, escribir y entrar |
| 7 | `rwx` (grupo) | El grupo puede leer, escribir y entrar |
| 0 | `---` (otros) | Nadie más tiene acceso |

En `ls -ld` el resultado se muestra como `drwxrws---`: la `s` en el permiso de grupo indica que `setgid` está activo.

### Bloque 5: SSH con claves

En este bloque se instala y activa el servicio SSH, se generan y copian las claves y se accede al servidor sin usar contraseña.

#### 5.1 Instalación y activación

Se instala el servicio SSH:

```bash
sudo apt install openssh-server
```

![Instalación del paquete openssh-server](recursos/imagenes/01-ubuntu-usuarios-ssh/instalacion-ssh.png)

Se habilita para que arranque con el sistema y se inicia de inmediato. En Ubuntu el servicio se llama `ssh`:

```bash
sudo systemctl enable --now ssh
sudo systemctl status ssh
```

![Servicio ssh habilitado y en estado active (running)](recursos/imagenes/01-ubuntu-usuarios-ssh/ssh-activado.png)

#### 5.2 Claves

Se genera un par de claves `ed25519` en la ubicación predeterminada `~/.ssh`. Esta primera captura corresponde al propio servidor (`admin01@srv-lab01`):

```bash
ssh-keygen -t ed25519
```

![Generación de un par de claves ed25519 en el servidor](recursos/imagenes/01-ubuntu-usuarios-ssh/keygen.png)

<!-- TODO: confirma si este par del servidor se usa en algún paso. En el resto del laboratorio la autenticación se hace con la clave generada en Windows (siguiente captura). -->

Para autenticarse desde el equipo anfitrión, la clave se genera en el cliente Windows. Se usa `ed25519` por ofrecer una seguridad alta con claves cortas y buen rendimiento. La clave privada (`id_ed25519`) no sale nunca del equipo local; solo se copia la pública (`id_ed25519.pub`).

```bat
ssh-keygen -t ed25519 -C "lab01-sysadmin"
```

![Generación del par de claves ed25519 en Windows](recursos/imagenes/01-ubuntu-usuarios-ssh/gen-clave-win.png)

Windows no incluye `ssh-copy-id`, así que la clave pública se copia a `~/.ssh/authorized_keys` de la VM con un comando ejecutado en el equipo local. Crea `~/.ssh` con permisos `700`, añade la clave (eliminando los saltos de línea de Windows con `tr -d '\r'`) y deja `authorized_keys` con permisos `600`:

```bat
type %USERPROFILE%\.ssh\id_ed25519.pub | ssh -p 2222 admin01@127.0.0.1 "mkdir -p ~/.ssh && chmod 700 ~/.ssh && tr -d '\r' >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
```

En esta primera conexión se acepta la huella del servidor y se pide la contraseña de `admin01` por última vez.

![Copia de la clave pública a authorized_keys de admin01](recursos/imagenes/01-ubuntu-usuarios-ssh/copia-clave.png)

Se prueba el acceso sin contraseña desde el equipo local:

```bat
ssh -p 2222 admin01@127.0.0.1
```

En la captura aparece, encima, la petición de contraseña de la copia anterior; el nuevo acceso entra directamente al prompt sin pedirla.

![Acceso SSH como admin01 sin solicitar la contraseña de la cuenta](recursos/imagenes/01-ubuntu-usuarios-ssh/ssh-no-passwd.png)

### Bloque 6: Endurecimiento de SSH

Este bloque restringe el acceso por SSH: se impide el login de `root` y se desactiva la autenticación por contraseña, de modo que solo se pueda entrar con clave pública.

#### 6.1 Copia de seguridad

Antes de modificar `sshd_config` se hace una copia de seguridad, para poder restaurar el archivo original si hay algún error:

```bash
sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak
ls /etc/ssh/
```

![Copia de seguridad sshd_config.bak](recursos/imagenes/01-ubuntu-usuarios-ssh/sshd-copia.png)

#### 6.2 Desactivar el login de root

Se edita `/etc/ssh/sshd_config` y se indica que `root` no puede iniciar sesión. Reduce la superficie de ataque, ya que `root` es el usuario que más prueban los ataques automatizados:

```
PermitRootLogin no
```

![PermitRootLogin no en sshd_config](recursos/imagenes/01-ubuntu-usuarios-ssh/root-no.png)

Se valida la sintaxis con `sshd -t` (sin salida si es correcta) y se reinicia el servicio:

```bash
sudo sshd -t
sudo systemctl restart ssh
sudo systemctl status ssh
```

![Validación con sshd -t y reinicio del servicio ssh](recursos/imagenes/01-ubuntu-usuarios-ssh/save-root-no.png)

Se prueba desde el equipo local a entrar como `root`, y el acceso es rechazado:

```bat
ssh -p 2222 root@127.0.0.1
```

![Intento de acceso como root rechazado](recursos/imagenes/01-ubuntu-usuarios-ssh/root-rechazado.png)

Se prueba también con un usuario que no es `root` (`user-01-dev`). En este punto la autenticación por contraseña sigue activa, por lo que el acceso es posible:

```bat
ssh -p 2222 user-01-dev@127.0.0.1
```

![Acceso con user-01-dev mediante contraseña, aún habilitada](recursos/imagenes/01-ubuntu-usuarios-ssh/ssh-dev.png)

#### 6.3 Desactivar las contraseñas

Con la clave ya probada, se desactiva la autenticación por contraseña para que solo se pueda entrar con clave pública. Esto elimina los ataques de fuerza bruta contra contraseñas. Dentro de `sshd_config`:

```
PasswordAuthentication no
```

![PasswordAuthentication no en sshd_config](recursos/imagenes/01-ubuntu-usuarios-ssh/pass-auth.png)

Se valida la sintaxis y se reinicia el servicio:

```bash
sudo sshd -t
sudo systemctl restart ssh
sudo systemctl status ssh
```

![Validación y reinicio del servicio ssh tras desactivar las contraseñas](recursos/imagenes/01-ubuntu-usuarios-ssh/save-pass-auth.png)

Se intenta iniciar sesión con `user-01-dev` desde el equipo local. El servidor ya solo admite clave pública y rechaza el acceso:

```bat
ssh -p 2222 user-01-dev@127.0.0.1
```

![Intento de login de user-01-dev rechazado: solo se admite publickey](recursos/imagenes/01-ubuntu-usuarios-ssh/pk-login.png)

### 7. Snapshot final

Se crea la última instantánea de la VM con la configuración terminada.

![Snapshot Final en VirtualBox](recursos/imagenes/01-ubuntu-usuarios-ssh/snapshot-final.png)

## Comprobación

| Qué se comprueba | Resultado | Evidencia |
|---|---|---|
| Hostname y zona horaria | `srv-lab01` y `Europe/Madrid` | [equipo-zona](recursos/imagenes/01-ubuntu-usuarios-ssh/equipo-zona.png) |
| Grupos creados | `dev` (GID 1002) y `soporte` (GID 1003) | [dev-soporte-groups](recursos/imagenes/01-ubuntu-usuarios-ssh/dev-soporte-groups.png) |
| Pertenencia a grupos | `user-01-dev` en `dev` y `user-01-soporte` en `soporte` | [anadir-users-grupo](recursos/imagenes/01-ubuntu-usuarios-ssh/anadir-users-grupo.png) |
| Servicio SSH | `active (running)` y habilitado | [ssh-activado](recursos/imagenes/01-ubuntu-usuarios-ssh/ssh-activado.png) |
| Acceso por clave | `admin01` entra sin pedir la contraseña de la cuenta | [ssh-no-passwd](recursos/imagenes/01-ubuntu-usuarios-ssh/ssh-no-passwd.png) |
| Login de `root` | Rechazado (`Permission denied`) | [root-rechazado](recursos/imagenes/01-ubuntu-usuarios-ssh/root-rechazado.png) |
| Autenticación por contraseña | Desactivada: `Permission denied (publickey)` | [pk-login](recursos/imagenes/01-ubuntu-usuarios-ssh/pk-login.png) |
| Instantánea final | Creada | [snapshot-final](recursos/imagenes/01-ubuntu-usuarios-ssh/snapshot-final.png) |

<!-- TODO (comprobaciones que faltan, haz la captura tú si las ejecutas):
 1. Permisos de los directorios compartidos: `ls -ld /srv/proyectos /srv/soporte` (debe verse drwxrws--- root dev / root soporte).
 2. Prueba de herencia de grupo: user-01-dev crea un fichero en /srv/proyectos y `ls -l` muestra el grupo dev; user-01-soporte recibe "Permission denied" al intentar entrar. (Esta es la comprobación que tenías en tu checklist, pero no tiene captura.)
 3. Valor efectivo de la configuración: `sudo sshd -T | grep -Ei 'permitrootlogin|passwordauthentication'`.
 Nota sobre el rechazo de root: en Ubuntu la cuenta root no tiene contraseña, así que ese intento falla aunque PermitRootLogin estuviera en yes. Por eso conviene la comprobación 3. -->

## Buenas prácticas aplicadas

| Práctica | Cómo se aplicó |
|---|---|
| No cerrarse la puerta | Dos terminales abiertas al desactivar la contraseña, para poder revertir si fallaba |
| Copia antes de editar | `sshd_config.bak` y validación con `sshd -t` antes de reiniciar |
| Instantáneas | Una tras la instalación y otra al terminar |
| Clave privada fuera del repositorio | `id_ed25519*` excluido en el `.gitignore` |
| Mínimo privilegio | Grupos `dev` y `soporte` con directorios independientes y sin acceso para otros |
| Sin trabajar como root | Usuario `admin01` con `sudo` |
| Permisos de `~/.ssh` | `700` en el directorio y `600` en `authorized_keys`; si son incorrectos, SSH ignora la clave |

## Problemas encontrados

### 1. `ssh-copy-id` no existe en Windows

- **Síntoma:** `"ssh-copy-id" no se reconoce como un comando interno o externo`.
- **Causa:** `ssh-copy-id` no está disponible en CMD ni en PowerShell.
- **Solución:** enviar la clave por tubería con `type ... | ssh ...`, como se describe en el apartado [5.2](#52-claves).

### 2. No se encontraba la ruta de la clave pública

- **Síntoma:** `El sistema no puede encontrar la ruta especificada` al ejecutar `type %USERPROFILE%\.ssh\id_ed25519.pub`.
- **Causa:** el par de claves aún no se había generado en el equipo local, por lo que no existía la carpeta `.ssh`.
- **Solución:** generar el par con `ssh-keygen -t ed25519` y comprobar con `dir %USERPROFILE%\.ssh`.

### 3. Conexión rechazada y reiniciada en el primer acceso por SSH

- **Síntoma:** primero `Connection refused` y después `kex_exchange_identification: read: Connection reset` / `Connection reset by 127.0.0.1 port 2222`.
- **Comprobaciones realizadas:** el servicio `ssh` estaba activo, escuchaba en el puerto 22 (`ss -tlnp | grep :22`) y se añadió la regla `ufw allow 22/tcp`.
- **Causa:** <!-- TODO: completar con la causa real -->
- **Solución:** <!-- TODO: completar con lo que lo arregló -->

### 4. Opción de configuración mal escrita en `sshd_config`

- **Síntoma:** `sudo sshd -t` mostraba `line 58: Bad configuration option: PermitLoginRoot`.
- **Causa:** errata en el nombre de la directiva (`PermitLoginRoot` en lugar de `PermitRootLogin`).
- **Solución:** corregir la línea 58 a `PermitRootLogin no` y volver a validar con `sudo sshd -t` hasta que no muestre errores. Validar antes de reiniciar evitó dejar el servicio SSH sin arrancar.

## Conclusiones

- Se obtiene un servidor con acceso administrativo por `sudo`, grupos de trabajo aislados y SSH que solo admite clave pública, sin login de `root`.
- Las copias de seguridad (`sshd_config.bak`) y las instantáneas permiten revertir cualquier cambio de configuración.
- Mejoras posibles: restringir el acceso con `ufw`, añadir `fail2ban` y automatizar esta configuración con Ansible en un laboratorio posterior.
