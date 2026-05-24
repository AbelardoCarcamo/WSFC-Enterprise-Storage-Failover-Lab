# 🖥️ Laboratorio WSFC — Clúster de File Server con Windows Server 2019, FreeNAS e iSCSI

![Windows Server](https://img.shields.io/badge/Windows_Server-2019-0078D6?style=for-the-badge&logo=windows&logoColor=white)
![FreeNAS](https://img.shields.io/badge/FreeNAS-iSCSI-003594?style=for-the-badge&logo=freenas&logoColor=white)
![VMware](https://img.shields.io/badge/VMware-Workstation_Pro-607078?style=for-the-badge&logo=vmware&logoColor=white)
![Status](https://img.shields.io/badge/Estado-Completado-brightgreen?style=for-the-badge)

---

## 📋 Tabla de Contenidos

- [Descripción General](#-descripción-general)
- [Arquitectura del Laboratorio](#-arquitectura-del-laboratorio)
- [Requisitos del Sistema](#-requisitos-del-sistema)
- [Topología de Red](#-topología-de-red)
- [Especificaciones de las VMs](#-especificaciones-de-las-vms)
- [Implementación](#-implementación)
  - [Fase 1 — Controlador de Dominio (DC01)](#fase-1--controlador-de-dominio-dc01)
  - [Fase 2 — Almacenamiento iSCSI (FreeNAS)](#fase-2--almacenamiento-iscsi-freenas)
  - [Fase 3 — Nodos del Clúster (SRV01 y SRV02)](#fase-3--nodos-del-clúster-srv01-y-srv02)
  - [Fase 4 — Failover Clustering](#fase-4--failover-clustering)
  - [Fase 5 — Quorum y Almacenamiento](#fase-5--quorum-y-almacenamiento)
  - [Fase 6 — Rol File Server](#fase-6--rol-file-server)
  - [Fase 7 — Prueba de Failover](#fase-7--prueba-de-failover)
- [Resultados](#-resultados)
- [Problemas Encontrados y Soluciones](#-problemas-encontrados-y-soluciones)
- [Tecnologías Utilizadas](#-tecnologías-utilizadas)
- [Autor](#-autor)

---

## 📖 Descripción General

Este laboratorio implementa un **clúster de alta disponibilidad (WSFC — Windows Server Failover Cluster)** para un servicio de File Server, utilizando almacenamiento compartido iSCSI provisto por FreeNAS. El objetivo principal es garantizar la continuidad del servicio ante la falla de uno de los nodos, mediante la migración automática de roles entre servidores.

Este proyecto fue desarrollado como parte de la asignatura de **Redes y Sistemas** en la **Universidad Tecnológica de Panamá**, ejecutado completamente en un entorno virtualizado con VMware Workstation Pro.

**Objetivos del laboratorio:**
- Configurar Active Directory como base de autenticación del clúster
- Provisionar almacenamiento compartido iSCSI mediante FreeNAS (ZFS + Zvols)
- Implementar Windows Server Failover Clustering (WSFC) con dos nodos
- Validar la alta disponibilidad del servicio mediante una prueba de failover real

---

## 🏗️ Arquitectura del Laboratorio

```
┌─────────────────────────────────────────────────────────────┐
│                   VMnet1 — 192.168.183.x                    │
│                                                             │
│  ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐  │
│  │  DC01   │    │  SRV01  │    │  SRV02  │    │ FreeNAS │  │
│  │  .10    │◄───│  .11    │    │  .12    │───►│  .128   │  │
│  │  AD DS  │    │  Node 1 │    │  Node 2 │    │  iSCSI  │  │
│  └─────────┘    └────┬────┘    └────┬────┘    └────┬────┘  │
│                      │              │               │       │
│                      └──────┬───────┘               │       │
│                             │                       │       │
│                    ┌────────▼────────┐              │       │
│                    │   CLUSTER-FS    │◄─────────────┘       │
│                    │  192.168.183.50 │  iSCSI LUN 0,1,2     │
│                    │                │                       │
│                    │   FS-CLUSTER   │                       │
│                    │  192.168.183.51│                       │
│                    └─────────────────┘                      │
│                                                             │
│  Host Windows (Cliente) — 192.168.183.1                     │
└─────────────────────────────────────────────────────────────┘
```

---

## ⚙️ Requisitos del Sistema

**Hardware del host:**
- CPU: 8 núcleos / 16 procesadores lógicos
- RAM: 16 GB
- VMware Workstation Pro instalado

**Software:**
- Windows Server 2019 (Desktop Experience)
- FreeNAS (TrueNAS Core)
- VMware Workstation Pro

---

## 🌐 Topología de Red

| VM | IP | Rol |
|----|----|-----|
| DC01 | 192.168.183.10 | Controlador de Dominio (AD DS) |
| SRV01 | 192.168.183.11 | Nodo 1 del clúster |
| SRV02 | 192.168.183.12 | Nodo 2 del clúster |
| FreeNAS | 192.168.183.128 | Almacenamiento iSCSI |
| CLUSTER-FS | 192.168.183.50 | IP del clúster |
| FS-CLUSTER | 192.168.183.51 | IP del rol File Server |
| Host | 192.168.183.1 | Cliente para pruebas |

**Red:** VMnet1 (Host-Only) — Máscara: `255.255.255.0`

---

## 🖥️ Especificaciones de las VMs

| VM | RAM | vCPU | Disco OS | Discos adicionales |
|----|-----|------|----------|--------------------|
| DC01 | 2 GB | 2 | 20 GB | — |
| SRV01 | 2.5 GB | 2 | 20 GB | — |
| SRV02 | 2.5 GB | 2 | 20 GB | — |
| FreeNAS | 2 GB | 2 | 10 GB | 2× 60 GB + 1× 1 GB |

**Almacenamiento iSCSI en FreeNAS (ZFS):**

| Pool | Zvol | Tamaño | LUN | Uso |
|------|------|--------|-----|-----|
| fs_disk1 | fs_disk1 | ~30 GB | LUN 0 | File Server |
| fs_disk2 | fs_disk2 | ~32 GB | LUN 1 | File Server |
| quorum | quorum | ~2 GB | LUN 2 | Quorum del clúster |

---

## 🔧 Implementación

### Fase 1 — Controlador de Dominio (DC01)

Se configuró DC01 como el primer servidor del entorno, asignando una IP estática y promoviendo el rol de Active Directory Domain Services para crear el dominio `cluster.local`.

**Pasos realizados:**
1. Asignación de IP estática: `192.168.183.10`, DNS: `127.0.0.1`
2. Instalación del rol AD DS
3. Promoción a controlador de dominio con el dominio `cluster.local`

| Captura | Descripción |
|---------|-------------|
| ![IP DC01](capturas/Configuracion%20de%20IP%20estatica%20a%20DC01.png) | Configuración de IP estática en DC01 |
| ![Dominio pt1](capturas/Creaci%C3%B3n%20de%20dominio%20pt.1.png) | Inicio del asistente de promoción AD DS |
| ![Dominio pt2](capturas/Creacion%20de%20dominio%20pt2.png) | Configuración del nombre de dominio |
| ![Nombre dominio](capturas/Nombramiento%20de%20dominio%20en%20dc01.png) | Asignación del nombre `cluster.local` |
| ![Confirmación dominio](capturas/Mensaje%20de%20confirmacion%20al%20crear%20el%20dominio%20en%20dc01.png) | Confirmación de la creación del dominio |
| ![Reinicio](capturas/Reinicio%20despues%20de%20hacer%20el%20server%20dominio.png) | Reinicio tras la promoción |

---

### Fase 2 — Almacenamiento iSCSI (FreeNAS)

FreeNAS se configuró como servidor de almacenamiento iSCSI con tres LUNs basados en Zvols ZFS. Se crearon pools separados para garantizar el aislamiento de los datos.

**Estructura de almacenamiento:**
```
Pool fs_disk1  →  Zvol fs_disk1  →  LUN 0  (File Server)
Pool fs_disk2  →  Zvol fs_disk2  →  LUN 1  (File Server)
Pool quorum    →  Zvol quorum    →  LUN 2  (Quorum)
```

**Configuración iSCSI:**
- Portal: `0.0.0.0:3260`
- Target: `iqn.2024-01.local.cluster:storage`
- Initiators: red `192.168.183.0/24`
- Logical Block Size: `512` en todos los extents

---

### Fase 3 — Nodos del Clúster (SRV01 y SRV02)

Ambos nodos fueron unidos al dominio `cluster.local` y conectados al target iSCSI de FreeNAS. Los discos compartidos fueron inicializados en GPT y formateados en NTFS desde SRV01.

| Captura | Descripción |
|---------|-------------|
| ![iSCSI SRV01](capturas/Activacion%20del%20protocolo%20ISCSI%20en%20SRV01.png) | Activación del iniciador iSCSI en SRV01 |
| ![Target conectado](capturas/Estado%20CONNECTED%20al%20target%20ISCSI%20del%20FREENAS.png) | Conexión exitosa al target de FreeNAS |
| ![Unir dominio](capturas/Unir%20SRV01%20al%20DOMAIN%20cluster.local.png) | Unión de SRV01 al dominio `cluster.local` |
| ![Discos sin asignar](capturas/DiskManagement%20Discos%20Unnallocated.png) | Discos iSCSI visibles en Administración de discos |
| ![Formateo GPT](capturas/Formateo%20de%20discos%20en%20GPT%20en%20DiskManagement.png) | Formateo de discos en GPT + NTFS |
| ![Discos saludables](capturas/Healthy%20disk%20ya%20formateados%20en%20DISKMANAGEMTN.png) | Discos formateados y saludables |
| ![SRV02 discos](capturas/Vista%20de%20los%20discos%20desde%20SRV02%20despues%20de%20unirlo%20al%20dominio.png) | Discos visibles desde SRV02 (Offline, gestionados por el clúster) |

---

### Fase 4 — Failover Clustering

Se instaló la característica **Failover Clustering** en ambos nodos y se creó el clúster. La creación mediante el asistente GUI falló por un timeout relacionado con el perfil de red NLA detectando `NoTraffic`. Se resolvió reiniciando el servicio NLA y ejecutando la creación vía PowerShell.

**Solución al problema de NLA:**
```powershell
Restart-Service NlaSvc -Force
```
Resultado: `IPv4Connectivity` pasó de `NoTraffic` a `LocalNetwork` con categoría `DomainAuthenticated`.

**Creación del clúster:**
```powershell
New-Cluster -Name CLUSTER-FS -Node SRV01,SRV02 -StaticAddress 192.168.183.50 -NoStorage
```

| Captura | Descripción |
|---------|-------------|
| ![Failover feature](capturas/Descarga%20del%20feature%20Failover%20Cluster%20en%20SRV01.png) | Instalación de Failover Clustering en SRV01 |
| ![cluadmin](capturas/Iniciar%20el%20msc%20cluadmin.png) | Apertura del Administrador de clústeres |
| ![Before you begin](capturas/Before%20you%20begin%20-%20Failover%20cluster%20message.png) | Asistente de configuración del clúster |
| ![Select SRV01](capturas/Select%20computers%20SRV01%20Failover%20cluster.png) | Selección de SRV01 como nodo |
| ![Select SRV02](capturas/Select%20computers%20SRV02%20Failover%20Cluster.png) | Selección de SRV02 como nodo |
| ![Credenciales](capturas/Validar%20credenciales%20al%20marcar%20SRV01%20Y%20SRV02%20en%20cluster7.png) | Validación de credenciales de dominio |
| ![Nodos seleccionados](capturas/Select%20servers%20or%20a%20cluster%20con%20los%20nodos%20seleccionados.png) | Ambos nodos agregados al asistente |
| ![Validación](capturas/Validating%20a%20configuration%20with%20the%20wizard.png) | Validación de configuración del clúster |
| ![PowerShell cluster](capturas/Creation%20of%20the%20cluster%20using%20PowerShell.png) | Creación exitosa del clúster vía PowerShell |
| ![Confirmación](capturas/Confirmation%20al%20crear%20Cluster.png) | Confirmación de la creación del clúster |
| ![Wizard confirm](capturas/Confirmation%20to%20create%20a%20cluster%20with%20the%20wizard.png) | Vista del asistente con el clúster creado |

---

### Fase 5 — Quorum y Almacenamiento

Los tres discos iSCSI fueron agregados al clúster como almacenamiento disponible. El disco de ~2 GB fue designado como **Quorum (testigo de disco)**.

```powershell
Get-ClusterAvailableDisk | Add-ClusterDisk
```

| Captura | Descripción |
|---------|-------------|
| ![Creation Quorum](capturas/Creation%20of%20QUORUM.png) | Inicio de la configuración del quorum |
| ![Select Quorum pt1](capturas/Select%20QUORUM%20configuration%20pt1..png) | Selección del tipo de quorum |
| ![Select witness](capturas/Select%20the%20quorum%20witness.png) | Selección del testigo de disco |
| ![Quorum disk](capturas/Cluster%20DISK%202%20QUORUM%202.5GB.png) | Cluster Disk 2 asignado como quorum |
| ![Get-ClusterQuorum](capturas/gETcLUSTER%20quorum.png) | Verificación del quorum por PowerShell |

---

### Fase 6 — Rol File Server

Se instaló el rol **Servidor de archivos** en SRV01 y SRV02 mediante el Administrador del servidor. Luego se configuró como rol del clúster con el nombre `FS-CLUSTER` y la IP `192.168.183.51`, creando la carpeta compartida `DatosCompartidos`.

| Captura | Descripción |
|---------|-------------|
| ![Configure Role](capturas/Configure%20Role.png) | Asistente de configuración de rol del clúster |
| ![Configure Role pt2](capturas/Configure%20role%20pt.2.png) | Configuración del nombre e IP del rol |
| ![File Server general](capturas/File%20Server%20for%20general%20use.png) | Selección del tipo de File Server |
| ![Select rol](capturas/Select%20rol%20-%20File%20Server.png) | Selección del rol File Server |
| ![Rol creado](capturas/Captura%20del%20rol%20de%20file%20server%20creado%20donde%20se%20ve%20el%20owner%20node%20siendo%20SRV01.png) | Rol FS-CLUSTER activo con SRV01 como propietario |

---

### Fase 7 — Prueba de Failover

Con el rol `FS-CLUSTER` activo en SRV01, se apagó el nodo para simular una falla. El clúster migró automáticamente el rol a SRV02 y el recurso compartido `DatosCompartidos` permaneció accesible desde el host.

| Captura | Descripción |
|---------|-------------|
| ![Apagar SRV01](capturas/Apagar%20SRV01%20para%20que%20SRV02%20lo%20reemplace.png) | Apagado de SRV01 para simular falla |
| ![Acceso sin SRV01](capturas/Con%20acceso%20a%20datoscompartidos%20aun%20sin%20SRV01%20activo.png) | Carpeta accesible con SRV01 apagado |
| ![Vista desde host](capturas/vISTZADO%20DE%20LA%20CARPETA%20DATOSCOMPARTIDOS%20DESDE%20EL%20HOST.png) | Acceso a `\\192.168.183.51\DatosCompartidos` desde el host |

---

## ✅ Resultados

| Componente | Estado |
|-----------|--------|
| DC01 — Active Directory (`cluster.local`) | ✅ Operativo |
| FreeNAS — iSCSI (3 LUNs vía ZFS Zvols) | ✅ Operativo |
| SRV01 — Nodo 1 del clúster | ✅ Online |
| SRV02 — Nodo 2 del clúster | ✅ Online |
| CLUSTER-FS — IP `192.168.183.50` | ✅ Activo |
| Quorum — Cluster Disk 2 | ✅ Configurado |
| FS-CLUSTER — IP `192.168.183.51` | ✅ En ejecución |
| Carpeta `DatosCompartidos` — acceso desde host | ✅ Accesible |
| Failover automático (SRV01 → SRV02) | ✅ Exitoso |

---

## 🐛 Problemas Encontrados y Soluciones

### 1. Discos iSCSI mostraban 16 KB en lugar del tamaño real
**Causa:** El *Logical Block Size* de los extents iSCSI estaba configurado incorrectamente.  
**Solución:** Cambiar el Logical Block Size a `512` y activar *Disable Physical Block Size* en cada extent de FreeNAS.

### 2. Timeout al crear el clúster desde la GUI
**Causa:** El servicio NLA detectaba la red como `IPv4Connectivity: NoTraffic`, lo que provocaba que Failover Clustering deshabilitara automáticamente el adaptador de red del clúster.  
**Solución:**
```powershell
Restart-Service NlaSvc -Force
New-Cluster -Name CLUSTER-FS -Node SRV01,SRV02 -StaticAddress 192.168.183.50 -NoStorage
```

### 3. Discos iSCSI en Online en SRV02 (conflicto de acceso)
**Causa:** Al conectar SRV02 al target iSCSI, los discos quedaron en estado Online en ambos nodos simultáneamente.  
**Solución:** Poner los discos en estado Offline en SRV02 desde Administración de discos. El clúster gestiona el acceso exclusivo.

### 4. Sesiones locales en lugar de sesiones de dominio
**Causa:** Las sesiones abiertas usaban `SRV01\Administrator` en lugar de `CLUSTER\Administrator`.  
**Solución:** Iniciar sesión con credenciales de dominio antes de ejecutar operaciones de clúster.

---

## 🛠️ Tecnologías Utilizadas

| Tecnología | Uso |
|-----------|-----|
| Windows Server 2019 | SO de los nodos y controlador de dominio |
| Active Directory Domain Services | Autenticación y gestión de objetos de clúster |
| Windows Server Failover Clustering (WSFC) | Motor de alta disponibilidad |
| FreeNAS / TrueNAS Core | Servidor de almacenamiento iSCSI |
| ZFS (Zvols + Pools) | Sistema de archivos para los LUNs |
| iSCSI | Protocolo de almacenamiento en red |
| VMware Workstation Pro | Plataforma de virtualización |
| PowerShell | Automatización y resolución de problemas |

---

## 👤 Autor

**Abelardo Cárcamo**  
Estudiante de Licenciatura en Ciberseguridad — Universidad Tecnológica de Panamá  
Fundador de [Nexium Security](https://nexium-security.tech)

[![LinkedIn](https://img.shields.io/badge/LinkedIn-abelardocb-0A66C2?style=for-the-badge&logo=linkedin&logoColor=white)](https://linkedin.com/in/abelardocb)
[![GitHub](https://img.shields.io/badge/GitHub-AbelardoCarcamo-181717?style=for-the-badge&logo=github&logoColor=white)](https://github.com/AbelardoCarcamo)

---

> **Nota:** Las capturas de pantalla se encuentran en la carpeta `capturas/` del repositorio. Para visualizarlas correctamente, asegúrate de subir las imágenes con los mismos nombres de archivo al repositorio.
