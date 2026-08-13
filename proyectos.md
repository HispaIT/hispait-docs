---
title: Provisionamiento Automatizado de Sistemas mediante Arquitectura PXE
description: Manual técnico para el despliegue e integración de FOG Project en Ubuntu Server 24.04 LTS con pfSense, enfocado en el provisionamiento automatizado por red (PXE) de imágenes Windows para entornos corporativos con o sin Active Directory.
published: true
date: 2026-08-13T08:42:48.096Z
tags: fog-project, pxe, pfsense, active-directory, windows, linux
editor: markdown
dateCreated: 2026-08-13T08:42:48.096Z
---

# Provisionamiento Automatizado de Sistemas mediante Arquitectura PXE

# Introducción

El presente documento describe el procedimiento de instalación, configuración y uso de una arquitectura de despliegue de equipos por red mediante **FOG Project**, orientada a entornos corporativos gestionados por **Grupo iNova**.

La solución permite realizar la instalación o reinstalación de equipos de forma centralizada, reduciendo tiempos de intervención técnica y estandarizando la configuración de los puestos de trabajo. Todo el procedimiento ha sido desplegado y validado en un entorno de laboratorio controlado.

La arquitectura contempla dos escenarios operativos diferenciados:

1. **Despliegue de equipos fuera de dominio:**
   * Equipos integrados en grupo de trabajo.
   * Entornos sin Active Directory.
   * Clientes que no requieren unión automática a dominio.
2. **Despliegue de equipos integrados en dominio:**
   * Equipos integrados en Active Directory.
   * Unión automática o posterior al dominio.
   * Uso de DNS corporativo y políticas de grupo (GPOs).

---

# Objetivos

El objetivo de este manual es recoger el procedimiento completo para:

* Instalar y configurar un servidor **FOG Project** sobre **Ubuntu Server 24.04 LTS**.
* Configurar una red de despliegue con **pfSense** actuando como gateway, firewall y servidor DHCP.
* Integrar la arquitectura con **Windows Server 2022 Active Directory** cuando el entorno lo requiera.
* Preparar y generalizar una imagen maestra de **Windows**.
* Capturar la imagen maestra desde FOG Project.
* Desplegar la imagen en equipos cliente mediante arranque por red PXE, soportando modos **BIOS Legacy** y **UEFI**.
* Diferenciar claramente el procedimiento de despliegue entre equipos fuera de dominio e integrados en dominio.

---

# Arquitectura empleada

## Máquinas empleadas

Para la implementación y validación de esta arquitectura se han utilizado los siguientes componentes:

| Rol | Sistema Operativo | Función |
| :--- | :--- | :--- |
| **Servidor FOG** | Ubuntu Server 24.04 LTS | Alojar FOG Project y el repositorio de imágenes base. |
| **Controlador de Dominio** | Windows Server 2022 | Servicios de Active Directory y DNS corporativo. |
| **Firewall / Router** | pfSense | Puerta de enlace (Gateway), firewall y servidor DHCP con opciones PXE. |
| **Imagen Maestra** | Windows 10 | Equipo referencia para la preparación y captura de la imagen base. |
| **Equipo de pruebas** | Windows 10 | Equipo cliente destino para validar la clonación y despliegue. |

---

## Direccionamiento de referencia

> **Nota:** Por motivos de seguridad y confidencialidad, los octetos finales de las direcciones IP del entorno de laboratorio se representan mediante el comodín `192.168.10.X`. En los despliegues reales de clientes, esta direccion debe adaptarse al direccionamiento de la red corporativa del cliente.

| Elemento | Dirección IP | Función |
| :--- | :--- | :--- |
| **pfSense LAN** | `192.168.10.X` | Puerta de enlace por defecto y servidor DHCP. |
| **Windows Server 2022** | `192.168.10.X` | Controlador de Dominio Active Directory y Servidor DNS. |
| **Servidor FOG** | `192.168.10.X` | FOG Project, servidor PXE/TFTP y almacenamiento de imágenes. |
| **Dominio AD** | `inova.local` | Dominio corporativo de Active Directory. |

---

# Esquema lógico de la solución

La infraestructura se organiza en tres zonas principales de red:

### 1. Zona de Control
Agrupa los servicios centrales encargados de coordinar el arranque y la autenticación:
* **Servidor FOG (`192.168.10.X`):** Gestiona la biblioteca de imágenes, proporciona el menú de arranque iPXE y coordina las tareas de captura y despliegue.
* **Controlador de Dominio (`192.168.10.X`):** Ofrece la resolución DNS interna y la gestión centralizada de identidades para entornos integrados en Active Directory.
* **pfSense (`192.168.10.X`):** Actúa como router/firewall principal y distribuye direcciones IP mediante  DHCP.

### 2. Zona de Distribución
Corresponde al segmento de red local `192.168.10.0/24` donde transitan las comunicaciones operativas:
* Peticiones e inspección de concesiones DHCP.
* Transferencia de binarios de arranque TFTP/HTTP.
* Tráfico de datos de alta velocidad durante la captura y clonación de imágenes.
* Consultas DNS y resolución de registros SRV de Active Directory.

### 3. Zona de Puestos de Trabajo
Engloba los equipos (físicos o virtuales) destinados a:
* Formar la plantilla de la imagen maestra mediante *Sysprep*.
* Recibir la clonación masiva o reinstalación de puestos.
* Incorporarse a un grupo de trabajo local o unirse al dominio `inova.local` tras el despliegue.

---

# Funcionamiento general del despliegue PXE

El flujo secuencial desde el encendido hasta el primer arranque es el siguiente:

1. **Petición de arranque:** El equipo cliente inicia el encendido enviando una solicitud PXE a través de la tarjeta de red.
2. **Concesión DHCP:** pfSense responde entregando dirección IP, máscara de subred, puerta de enlace, DNS y los parámetros específicos PXE (Next-Server y Boot File Name).
3. **Carga de binario:** El cliente contacta con el Servidor FOG (`192.168.10.X`) para descargar e inicializar el menú de arranque iPXE.
4. **Ejecución de tareas:** FOG identifica el equipo y ejecuta la tarea asignada desde el panel web (Registro, Captura o Despliegue).
5. **Finalización:** Concluida la transferencia de la imagen, el cliente reinicia e inicia el sistema operativo desde el disco local.
6. **Post-configuración:** Dependiendo de la plantilla, el equipo permanece en grupo de trabajo o se une automáticamente al dominio.