<div align="center">

# WSFC Enterprise Storage Failover Lab

![Windows Server](https://img.shields.io/badge/Windows_Server-2019-0078D6?style=for-the-badge&logo=windows&logoColor=white)
![FreeNAS](https://img.shields.io/badge/TrueNAS-iSCSI_Storage-003594?style=for-the-badge&logo=truenas&logoColor=white)
![VMware](https://img.shields.io/badge/VMware-Workstation_Pro-607078?style=for-the-badge&logo=vmware&logoColor=white)
![PowerShell](https://img.shields.io/badge/PowerShell-5.1-5391FE?style=for-the-badge&logo=powershell&logoColor=white)
![Status](https://img.shields.io/badge/Estado-Completado-00C853?style=for-the-badge)

**Implementación de un clúster de alta disponibilidad con Windows Server Failover Clustering, almacenamiento compartido iSCSI sobre ZFS y prueba de failover en entorno virtualizado.**

</div>

---

## Tabla de Contenidos

- [Descripción General](#descripción-general)
- [Arquitectura](#arquitectura)
- [Infraestructura](#infraestructura)
- [Implementación](#implementación)
  - [Fase 1 — Controlador de Dominio](#fase-1--controlador-de-dominio-dc01)
  - [Fase 2 — Almacenamiento iSCSI](#fase-2--almacenamiento-iscsi-freenas)
  - [Fase 3 — Nodos del Clúster](#fase-3--nodos-del-clúster)
  - [Fase 4 — Failover Clustering](#fase-4--failover-clustering)
  - [Fase 5 — Quorum](#fase-5--quorum-y-almacenamiento-compartido)
  - [Fase 6 — Rol File Server](#fase-6--rol-file-server)
  - [Fase 7 — Prueba de Failover](#fase-7--prueba-de-failover)
- [Resultados](#resultados)
- [Troubleshooting](#troubleshooting)
- [Stack Tecnológico](#stack-tecnológico)

---

## Descripción General

Este laboratorio implementa un **clúster de alta disponibilidad (WSFC)** para un servicio de File Server con almacenamiento compartido iSCSI provisto por FreeNAS/TrueNAS Core sobre ZFS. El entorno replica una arquitectura de producción empresarial con dos nodos de cómputo, controlador de dominio, almacenamiento en red y prueba de failover real.

**Objetivos técnicos:**

- Provisionar almacenamiento compartido iSCSI con ZFS Zvols desde FreeNAS
- Configurar Active Directory como base de autenticación del clúster
- Desplegar Windows Server Failover Clustering (WSFC) con dos nodos activos
- Validar continuidad del servicio mediante failover automático entre nodos

---

## Arquitectura

```
┌──────────────────────────────────────────────────────────────────┐
│                      VMnet1 · 192.168.183.x                      │
│                                                                  │
│   ┌──────────┐     ┌──────────┐     ┌──────────┐                │
│   │   DC01   │     │  SRV01   │     │  SRV02   │                │
│   │  .10     │◄────│  .11     │     │  .12     │                │
│   │  AD DS   │     │  Node 1  │     │  Node 2  │                │
│   └──────────┘     └────┬─────┘     └────┬─────┘                │
│                         │                │                       │
│              iSCSI LUN 0,1,2             │                       │
│                         │                │                       │
│                    ┌────▼────────────────▼────┐                  │
│                    │        FreeNAS .128       │                  │
│                    │  Pool fs_disk1 · LUN 0    │                  │
│                    │  Pool fs_disk2 · LUN 1    │                  │
│                    │  Pool quorum   · LUN 2    │                  │
│                    └───────────────────────────┘                  │
│                                                                  │
│        ┌─────────────────────────────────────┐                   │
│        │           CLUSTER-FS .50            │                   │
│        │     FS-CLUSTER (File Server) .51    │                   │
│        │      \\192.168.183.51\DatosCompartidos │                 │
│        └─────────────────────────────────────┘                   │
│                                                                  │
│   Host Windows (Cliente de pruebas) · 192.168.183.1             │
└──────────────────────────────────────────────────────────────────┘
```

---

## Infraestructura

### Topología de Red

| Nodo | IP | Rol |
|------|-----|-----|
| DC01 | `192.168.183.10` | Controlador de Dominio · AD DS |
| SRV01 | `192.168.183.11` | Nodo 1 del clúster |
| SRV02 | `192.168.183.12` | Nodo 2 del clúster |
| FreeNAS | `192.168.183.128` | Servidor de almacenamiento iSCSI |
| CLUSTER-FS | `192.168.183.50` | IP virtual del clúster |
| FS-CLUSTER | `192.168.183.51` | IP virtual del rol File Server |

> Red: **VMnet1 (Host-Only)** · Subred: `255.255.255.0` · Dominio: `cluster.local`

### Especificaciones de VMs

| VM | RAM | vCPU | Disco OS |
|----|-----|------|----------|
| DC01 | 2 GB | 2 | 20 GB |
| SRV01 | 2.5 GB | 2 | 20 GB |
| SRV02 | 2.5 GB | 2 | 20 GB |
| FreeNAS | 2 GB | 2 | 10 GB + 2×60 GB + 1×1 GB |

### Almacenamiento iSCSI (ZFS)

| Pool | Zvol | LUN | Tamaño | Uso |
|------|------|-----|--------|-----|
| fs_disk1 | fs_disk1 | LUN 0 | ~30 GB | File Server |
| fs_disk2 | fs_disk2 | LUN 1 | ~32 GB | File Server |
| quorum | quorum | LUN 2 | ~2 GB | Quorum del clúster |

> Configuración iSCSI: Portal `0.0.0.0:3260` · Target `iqn.2024-01.local.cluster:storage` · Block Size `512`

---

## Implementación

### Fase 1 — Controlador de Dominio (DC01)

Configuración de DC01 con IP estática y promoción del rol AD DS para establecer el dominio `cluster.local`, base de autenticación de todo el entorno.

| | |
|---|---|
| ![IP DC01](Capturas/Configuracion%20de%20IP%20estatica%20a%20DC01.png) | ![Dominio pt1](Capturas/Creaci%C3%B3n%20de%20dominio%20pt.1.png) |
| *Configuración de IP estática en DC01* | *Inicio del asistente de promoción AD DS* |
| ![Dominio pt2](Capturas/Creacion%20de%20dominio%20pt2.png) | ![Nombre dominio](Capturas/Nombramiento%20de%20dominio%20en%20dc01.png) |
| *Configuración del dominio* | *Asignación del nombre `cluster.local`* |
| ![Confirmación dominio](Capturas/Mensaje%20de%20confirmacion%20al%20crear%20el%20dominio%20en%20dc01.png) | ![Reinicio](Capturas/Reinicio%20despues%20de%20hacer%20el%20server%20dominio.png) |
| *Confirmación de la creación del dominio* | *Reinicio tras la promoción* |

---

### Fase 2 — Almacenamiento iSCSI (FreeNAS)

FreeNAS fue configurado con tres pools ZFS independientes, cada uno con un Zvol dedicado expuesto como LUN iSCSI. El Logical Block Size fue establecido en `512` para garantizar la correcta detección del tamaño de disco en los nodos Windows.

```
Pool fs_disk1  →  Zvol fs_disk1  →  extent_fs1    →  LUN 0
Pool fs_disk2  →  Zvol fs_disk2  →  extent_fs2    →  LUN 1
Pool quorum    →  Zvol quorum    →  extent_quorum  →  LUN 2
```

---

### Fase 3 — Nodos del Clúster

Ambos nodos fueron unidos al dominio `cluster.local` y conectados al target iSCSI. Los discos compartidos fueron inicializados en **GPT** y formateados en **NTFS** desde SRV01. SRV02 los ve en estado **Offline** — el clúster gestiona el acceso exclusivo.

| | |
|---|---|
| ![iSCSI SRV01](Capturas/Activacion%20del%20protocolo%20ISCSI%20en%20SRV01.png) | ![Target conectado](Capturas/Estado%20CONNECTED%20al%20target%20ISCSI%20del%20FREENAS.png) |
| *Activación del iniciador iSCSI en SRV01* | *Conexión exitosa al target de FreeNAS* |
| ![Unir dominio](Capturas/Unir%20SRV01%20al%20DOMAIN%20cluster.local.png) | ![Discos sin asignar](Capturas/DiskManagement%20Discos%20Unnallocated.png) |
| *Unión de SRV01 al dominio `cluster.local`* | *Discos iSCSI detectados en Disk Management* |
| ![Formateo GPT](Capturas/Formateo%20de%20discos%20en%20GPT%20en%20DiskManagement.png) | ![Discos saludables](Capturas/Healthy%20disk%20ya%20formateados%20en%20DISKMANAGEMTN.png) |
| *Formateo en GPT + NTFS* | *Discos formateados y saludables* |
| ![SRV02 discos](Capturas/Vista%20de%20los%20discos%20desde%20SRV02%20despues%20de%20unirlo%20al%20dominio.png) | |
| *Discos visibles desde SRV02 en estado Offline* | |

---

### Fase 4 — Failover Clustering

Instalación de la característica **Failover Clustering** en ambos nodos, validación de la configuración y creación del clúster. El asistente GUI falló por timeout por un problema de perfil de red NLA; se resolvió reiniciando el servicio y ejecutando la creación vía PowerShell.

> **Causa raíz:** NLA detectaba `IPv4Connectivity: NoTraffic`, lo que hacía que WSFC deshabilitara automáticamente la red del clúster.

```powershell
# Fix: reiniciar NLA para que detecte correctamente la red de dominio
Restart-Service NlaSvc -Force

# Crear el clúster directamente con PowerShell
New-Cluster -Name CLUSTER-FS -Node SRV01,SRV02 -StaticAddress 192.168.183.50 -NoStorage
```

| | |
|---|---|
| ![Failover feature](Capturas/Descarga%20del%20feature%20Failover%20Cluster%20en%20SRV01.png) | ![cluadmin](Capturas/Iniciar%20el%20msc%20cluadmin.png) |
| *Instalación de Failover Clustering en SRV01* | *Apertura del Administrador de clústeres* |
| ![Before you begin](Capturas/Before%20you%20begin%20-%20Failover%20cluster%20message.png) | ![Select SRV01](Capturas/Select%20computers%20SRV01%20Failover%20cluster.png) |
| *Asistente de configuración* | *Selección de SRV01 como nodo* |
| ![Select SRV02](Capturas/Select%20computers%20SRV02%20Failover%20Cluster.png) | ![Credenciales](Capturas/Validar%20credenciales%20al%20marcar%20SRV01%20Y%20SRV02%20en%20cluster7.png) |
| *Selección de SRV02 como nodo* | *Validación de credenciales de dominio* |
| ![Nodos seleccionados](Capturas/Select%20servers%20or%20a%20cluster%20con%20los%20nodos%20seleccionados.png) | ![Validación](Capturas/Validating%20a%20configuration%20with%20the%20wizard.png) |
| *Ambos nodos en el asistente* | *Validación de configuración* |
| ![PowerShell cluster](Capturas/Creation%20of%20the%20cluster%20using%20PowerShell.png) | ![Confirmación](Capturas/Confirmation%20al%20crear%20Cluster.png) |
| *Creación exitosa vía PowerShell* | *Clúster CLUSTER-FS creado* |
| ![Wizard confirm](Capturas/Confirmation%20to%20create%20a%20cluster%20with%20the%20wizard.png) | |
| *Vista final del clúster en cluadmin* | |

---

### Fase 5 — Quorum y Almacenamiento Compartido

Los tres discos iSCSI fueron agregados al clúster. El disco de ~2 GB fue designado como **testigo de disco (Quorum)** para garantizar la resolución de split-brain.

```powershell
Get-ClusterAvailableDisk | Add-ClusterDisk
```

| | |
|---|---|
| ![Creation Quorum](Capturas/Creation%20of%20QUORUM.png) | ![Select Quorum pt1](Capturas/Select%20QUORUM%20configuration%20pt1..png) |
| *Inicio de la configuración del quorum* | *Selección del tipo de quorum* |
| ![Select witness](Capturas/Select%20the%20quorum%20witness.png) | ![Quorum disk](Capturas/Cluster%20DISK%202%20QUORUM%202.5GB.png) |
| *Selección del testigo de disco* | *Cluster Disk 2 asignado como quorum* |
| ![Get-ClusterQuorum](Capturas/gETcLUSTER%20quorum.png) | |
| *Verificación del quorum por PowerShell* | |

---

### Fase 6 — Rol File Server

Instalación del rol **Servidor de archivos** en ambos nodos y configuración como rol de alta disponibilidad del clúster con nombre `FS-CLUSTER`, IP `192.168.183.51` y recurso compartido `DatosCompartidos`.

| | |
|---|---|
| ![Configure Role](Capturas/Configure%20Role.png) | ![Configure Role pt2](Capturas/Configure%20role%20pt.2.png) |
| *Asistente de configuración de rol de clúster* | *Configuración del nombre e IP del rol* |
| ![File Server general](Capturas/File%20Server%20for%20general%20use.png) | ![Select rol](Capturas/Select%20rol%20-%20File%20Server.png) |
| *Tipo: Servidor de archivos para uso general* | *Selección del rol File Server* |
| ![Rol creado](Capturas/Captura%20del%20rol%20de%20file%20server%20creado%20donde%20se%20ve%20el%20owner%20node%20siendo%20SRV01.png) | |
| *Rol FS-CLUSTER activo · Owner: SRV01* | |

---

### Fase 7 — Prueba de Failover

Con el rol `FS-CLUSTER` activo en SRV01, se apagó el nodo para simular una falla real. El clúster detectó la interrupción y migró el rol automáticamente a SRV02. El recurso `\\192.168.183.51\DatosCompartidos` permaneció accesible desde el host sin intervención manual.

| | |
|---|---|
| ![Apagar SRV01](Capturas/Apagar%20SRV01%20para%20que%20SRV02%20lo%20reemplace.png) | ![Acceso sin SRV01](Capturas/Con%20acceso%20a%20datoscompartidos%20aun%20sin%20SRV01%20activo.png) |
| *Apagado de SRV01 para simular falla* | *Servicio activo con SRV01 apagado* |
| ![Vista desde host](Capturas/vISTZADO%20DE%20LA%20CARPETA%20DATOSCOMPARTIDOS%20DESDE%20EL%20HOST.png) | |
| *Acceso a `\\192.168.183.51\DatosCompartidos` desde el host* | |

---

## Resultados

| Componente | Estado |
|-----------|:------:|
| DC01 — Active Directory `cluster.local` | ✅ |
| FreeNAS — 3 LUNs iSCSI sobre ZFS | ✅ |
| SRV01 — Nodo 1 del clúster | ✅ |
| SRV02 — Nodo 2 del clúster | ✅ |
| CLUSTER-FS — IP `192.168.183.50` | ✅ |
| Quorum — Cluster Disk 2 | ✅ |
| FS-CLUSTER — IP `192.168.183.51` | ✅ |
| `\\192.168.183.51\DatosCompartidos` accesible | ✅ |
| Failover automático SRV01 → SRV02 | ✅ |

---

## Troubleshooting

### Discos iSCSI mostraban 16 KB en lugar del tamaño real
**Causa:** Logical Block Size incorrecto en los extents de FreeNAS.  
**Fix:** Establecer `Logical Block Size: 512` y activar `Disable Physical Block Size` en cada extent.

### Timeout al crear el clúster desde la GUI
**Causa:** El servicio NLA detectaba `IPv4Connectivity: NoTraffic`, haciendo que WSFC deshabilitara `Cluster Network 1` automáticamente.  
**Fix:**
```powershell
Restart-Service NlaSvc -Force
New-Cluster -Name CLUSTER-FS -Node SRV01,SRV02 -StaticAddress 192.168.183.50 -NoStorage
```

### Conflicto de acceso a discos en SRV02
**Causa:** Al conectar SRV02 al target iSCSI, los discos quedaron en estado Online en ambos nodos simultáneamente.  
**Fix:** Poner los discos en `Offline` en SRV02 desde Disk Management. El clúster gestiona el acceso exclusivo.

### Sesiones locales en lugar de sesiones de dominio
**Causa:** PowerShell y las operaciones de clúster se ejecutaban como `SRV01\Administrator` en lugar de `CLUSTER\Administrator`.  
**Fix:** Iniciar sesión con credenciales de dominio antes de cualquier operación de clúster.

---

## Stack Tecnológico

| Tecnología | Rol en el laboratorio |
|-----------|----------------------|
| Windows Server 2019 | SO de nodos y controlador de dominio |
| Active Directory Domain Services | Autenticación y gestión de objetos de clúster |
| Windows Server Failover Clustering | Motor de alta disponibilidad |
| FreeNAS / TrueNAS Core | Servidor de almacenamiento iSCSI |
| ZFS (Zvols + Pools) | Sistema de archivos de los LUNs |
| iSCSI | Protocolo de almacenamiento en red |
| VMware Workstation Pro | Plataforma de virtualización |
| PowerShell | Automatización y resolución de problemas |

---

<div align="center">

**Abelardo Cárcamo**

[![LinkedIn](https://img.shields.io/badge/LinkedIn-abelardocb-0A66C2?style=flat-square&logo=linkedin&logoColor=white)](https://linkedin.com/in/abelardocb)
[![GitHub](https://img.shields.io/badge/GitHub-AbelardoCarcamo-181717?style=flat-square&logo=github&logoColor=white)](https://github.com/AbelardoCarcamo)
[![Nexium Security](https://img.shields.io/badge/Nexium_Security-nexium--security.tech-00C853?style=flat-square)](https://nexium-security.tech)

</div>
