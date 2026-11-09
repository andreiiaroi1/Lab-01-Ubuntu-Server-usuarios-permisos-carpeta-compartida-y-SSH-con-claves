# Descripción del Documento: Lab 01 · Ubuntu Server

## 📌 Resumen General

El documento proporcionado es la documentación técnica y guía paso a paso del laboratorio **"Lab 01 · Ubuntu Server: usuarios, permisos, carpeta compartida y SSH con claves"**. Registra el despliegue, configuración y fortificación (*hardening*) inicial de un servidor Linux Ubuntu Server 26.04.1 LTS sobre Oracle VirtualBox.

## 🎯 Objetivos Principales del Laboratorio

1. **Despliegue y Red:** Instalar un servidor Ubuntu Server en entorno virtualizado con doble interfaz de red (Red Interna estática + NAT con port forwarding).

2. **Gestión de Usuarios y Accesos:** Implementar el principio de mínimo privilegio creando un usuario administrador (`admin01`) con `sudo`, prescindiendo del uso directo de `root`, y usuarios segregados en grupos específicos (`dev` y `soporte`).

3. **Control de Acceso y Colaboración:** Configurar directorios de trabajo colaborativo (`/srv/proyectos` y `/srv/soporte`) aplicando permisos estrictos (`2770`) y el bit especial `setgid` para garantizar la herencia de grupos en archivos nuevos.

4. **Endurecimiento (*Hardening*) de SSH:**

   * Autenticación exclusivamente mediante par de claves criptográficas (`ed25519`).

   * Deshabilitación total del acceso remoto del usuario `root` (`PermitRootLogin no`).

   * Desactivación de la autenticación por contraseña (`PasswordAuthentication no`).

## 🏗️ Arquitectura y Entorno Técnico

| Componente | Especificación / Valor | 
| ----- | ----- | 
| **Sistema Operativo** | Ubuntu Server 26.04.1 LTS | 
| **Virtualización** | Oracle VirtualBox (4 GB RAM, 2 vCPUs) | 
| **Hostname** | `srv-lab01` | 
| **Red Interna** | Interfaz `enp0s3` (IP Estática: `192.168.10.0/24`) | 
| **Red NAT** | Interfaz `enp0s8` (DHCP) + Reenvío de puerto host (`127.0.0.1:2222` → `VM:22`) | 
| **Algoritmo SSH** | Ed25519 (`ssh-keygen -t ed25519`) | 

## 📑 Estructura del Documento

El archivo `README.md` se compone de 8 secciones estructuradas:

1. **Objetivo:** Detalla las metas técnicas del ejercicio.

2. **Habilidades demostradas:** Resumen de competencias ejecutadas (gestión de grupos, permisos `setgid`, hardening de SSH, uso de snapshots).

3. **Requisitos y entorno:** Tablas resumen con la topología, datos de red, usuarios creados y rutas críticas del sistema (`/etc/netplan/`, `/etc/ssh/sshd_config`, etc.).

4. **Pasos (Desarrollo del Lab):**

   * **Bloque 1:** Instalación del SO y red con Netplan.

   * **Bloque 2:** Actualización del sistema (`apt update/upgrade`), timezone, hostname y snapshot inicial.

   * **Bloque 3:** Creación de usuarios (`admin01`, `user-01-dev`, `user-01-soporte`) y grupos (`dev`, `soporte`).

   * **Bloque 4:** Creación de directorios compartidos y aplicación de permisos `2770` (`drwxrws---`).

   * **Bloque 5:** Configuración de SSH, generación de claves en cliente Windows y copia mediante *piping* en CMD.

   * **Bloque 6:** Hardening del archivo `sshd_config` (desactivar `root` y contraseñas).

   * **Bloque 7:** Snapshot final de la VM.

5. **Comprobación:** Matriz de verificación de los resultados esperados frente a la evidencia obtenida.

6. **Buenas prácticas aplicadas:** Resumen de medidas preventivas tomadas durante el proceso (respaldos `.bak`, sesiones duplicadas antes de aplicar cambios de SSH, validación sintáctica con `sshd -t`).

7. **Problemas encontrados:** Bitácora de troubleshooting con errores reales (ausencia de `ssh-copy-id` en Windows CMD, sintaxis errónea en `sshd_config`, etc.) y sus respectivas soluciones.

8. **Conclusiones:** Balance del ejercicio y líneas de mejora futuras (integración de `ufw`, `fail2ban` o automatización con Ansible).
