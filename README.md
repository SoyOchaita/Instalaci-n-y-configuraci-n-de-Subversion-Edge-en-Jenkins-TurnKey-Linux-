# Proyecto Ingeniería de Software 2 – Fase 1  
## Instalación y configuración de Subversion Edge en Jenkins (TurnKey Linux) sin conexión a Internet

**Autor:** Nicolás Ochaita  
**Créditos:** Imagen Docker **`svnedge/app`** publicada por *svnedge* en [Docker Hub](https://hub.docker.com/r/svnedge/app/tags).  
Sin esta imagen pública no sería posible replicar este entorno. Todos los derechos y licencias pertenecen a su autor original.

---

## 🎯 Objetivo

Instalar y configurar **Subversion Edge (SVN Edge)** dentro de una máquina virtual **TurnKey Jenkins (Debian 12 “Bookworm”)** sin conexión a Internet, usando **Docker offline**, y dejar el sistema funcional con el repositorio del proyecto **Calculadora** accesible desde Eclipse mediante el plugin **Subclipse**.

---

## 🧱 Entorno utilizado

- **Sistema operativo:** TurnKey Jenkins (basado en Debian 12 Bookworm)  
- **Gestor web:** Webmin (interfaz administrativa del sistema)  
- **Modo de trabajo:** Todo se realizó **desde la terminal integrada en Webmin**  
  > No se utilizó `sudo` porque Webmin ejecuta comandos directamente como **root**.  
- **Contenedor:** `svnedge/app` (imagen oficial publicada en Docker Hub)  
- **Acceso sin Internet:** Todos los paquetes y la imagen Docker fueron **subidos manualmente a la VM mediante Webmin**.

---

## 📦 Archivos requeridos antes de iniciar

En la carpeta `/root/` de la máquina Jenkins deben existir los siguientes elementos:

```
/root/
 ├── docker-offline/
 │    ├── containerd.io_1.7.28-1~debian.12~bookworm_amd64.deb
 │    ├── docker-ce_5%3a28.5.1-1~debian.12~bookworm_amd64.deb
 │    ├── docker-ce-cli_5%3a28.5.1-1~debian.12~bookworm_amd64.deb
 │    ├── docker-buildx-plugin_0.29.1-1~debian.12~bookworm_amd64.deb
 │    └── docker-compose-plugin_2.40.0-1~debian.12~bookworm_amd64.deb
 └── svnedge-app.tar.gz
```

📤 Todos estos archivos fueron **subidos manualmente** por Webmin  
(`Others → File Manager → Upload to current directory`).

---

## 🪄 Procedimiento completo paso a paso

### 1️⃣ Descomprimir la imagen Docker de Subversion Edge

```bash
cd /root
gunzip svnedge-app.tar.gz
```

Esto genera el archivo:
```
svnedge-app.tar
```

---

### 2️⃣ Instalar Docker de forma offline

```bash
cd /root/docker-offline
dpkg -i *.deb
systemctl enable docker
systemctl start docker
```

Verificar instalación:

```bash
docker --version
```
> Si muestra la versión de Docker, la instalación fue exitosa.

---

### 3️⃣ Cargar la imagen `svnedge/app`

```bash
cd /root
docker load -i svnedge-app.tar
docker images
```
Deberías ver algo como:
```
REPOSITORY     TAG       IMAGE ID       SIZE
svnedge/app    latest    9e077ed8a6df   590MB
```

---

### 4️⃣ Crear y ejecutar el contenedor

Primero elimina versiones anteriores del contenedor (si existían):
```bash
docker rm -f svnedge
```

Luego crea uno nuevo publicando los puertos necesarios:

```bash
docker run -d \
  -p 3344:3343 \
  -p 18180:18080 \
  --restart always \
  --name svnedge \
  svnedge/app
```

📌 **Descripción de puertos:**
| Puerto interno | Puerto externo | Función |
|----------------|----------------|----------|
| 3343 | 3344 | Servicio Subversion (repositorios SVN) |
| 18080 | 18180 | Interfaz web y navegador ViewVC |

Verifica que esté activo:
```bash
docker ps
```
Salida esperada:
```
PORTS
0.0.0.0:3344->3343/tcp, 0.0.0.0:18180->18080/tcp
```

---

### 5️⃣ Acceder a la interfaz web

Desde un navegador en tu equipo (mismo segmento de red que la VM):

- **Panel de administración:**  
  `http://IP_DE_LA_VM:18180`
- **Navegador de código (ViewVC):**  
  `http://IP_DE_LA_VM:18180/viewvc`
- **Repositorio directo (para Eclipse/Subclipse):**  
  `http://IP_DE_LA_VM:3344/svn/Calculadora`

> Reemplaza `IP_DE_LA_VM` por la IP real de tu máquina Jenkins (por ejemplo `192.168.1.36`).

---

### 6️⃣ Configurar Subversion Edge

1. Accede al panel web (`http://IP_DE_LA_VM:18180`)  
2. Crea el usuario **admin** cuando lo pida el asistente inicial  
3. Acepta la licencia y finaliza la configuración  
4. Verifica que el **Subversion status** aparezca como **Up**  
5. En el menú lateral, selecciona **Repositories → Create**  
6. Escribe el nombre del repositorio:  
   ```
   Calculadora
   ```
7. Da clic en **Create**
8. Asegúrate de que el estado sea **OK**

La URL del repositorio será:
```
http://IP_DE_LA_VM:3344/svn/Calculadora
```

---

### 7️⃣ Probar conexión desde la terminal

```bash
svn list http://IP_DE_LA_VM:3344/svn/Calculadora --username admin
```
Debe pedir la contraseña y devolver un listado vacío (repositorio nuevo).

---

### 8️⃣ Conectar desde Eclipse (Subclipse)

1. Instalar **Subclipse** desde Marketplace  
2. Abrir la perspectiva **SVN Repository Exploring**  
3. Agregar nueva ubicación con:  
   - URL: `http://IP_DE_LA_VM:3344/svn/Calculadora`  
   - Usuario: `admin`  
   - Contraseña: *(la que definiste)*  
4. Crear el proyecto Java **Calculadora**  
5. Estructura mínima exigida por el curso:
```
Calculadora/
 ├── src/gt/edu/umes/proy/
 │    ├── Calculadora.java        # Clase con método main()
 │    └── LogicaDelNegocio.java   # Funciones suma y promedio
 ├── .classpath
 ├── .project
 └── README.md
```
6. Compartirlo con SVN:  
   **Team → Share Project → SVN → Commit inicial**

---

## 🧰 Solución de problemas comunes

| Problema | Causa probable | Solución |
|-----------|----------------|-----------|
| No abre `/viewvc` o `/svn/Calculadora` | El puerto `18080` interno no estaba publicado | Recrear contenedor con `-p 18180:18080` |
| “address already in use” | Puerto ya ocupado (por Jenkins) | Usa otro puerto, ej. `-p 18181:18080` |
| `DNS_PROBE_FINISHED_NXDOMAIN` | Intentas acceder al nombre interno del contenedor | Usa la IP de la máquina Jenkins |
| Panel no carga | Docker no iniciado | `systemctl start docker` y verifica con `docker ps` |
| Repos desaparecen al borrar contenedor | Falta de volumen persistente | Agregar `-v /var/svn-data:/var/opt/csvn` al comando `docker run` |

---

## 🧱 Recomendación para persistencia

Si se desea que los repositorios sobrevivan reinicios o recreaciones del contenedor:

```bash
mkdir -p /var/svn-data
docker rm -f svnedge
docker run -d \
  -p 3344:3343 \
  -p 18180:18080 \
  -v /var/svn-data:/var/opt/csvn \
  --restart always \
  --name svnedge \
  svnedge/app
```

---

## 📜 Créditos y licencia

- **Imagen oficial Docker:** [`svnedge/app`](https://hub.docker.com/r/svnedge/app/tags)  
  Publicada por **svnedge** — todos los derechos pertenecen a su autor original.  
- **Software Subversion Edge:** desarrollado por *CollabNet, Inc.*  
- Esta guía fue creada con fines **educativos y demostrativos** en la Universidad Mesoamericana – UMES 2025.

---

## ✅ Checklist de verificación

- [ ] Docker instalado correctamente (offline)  
- [ ] Imagen `svnedge/app` cargada con éxito  
- [ ] Contenedor ejecutándose (`docker ps`)  
- [ ] Panel accesible en `http://IP_VM:18180`  
- [ ] ViewVC funcionando (`/viewvc`)  
- [ ] Repositorio `Calculadora` creado y visible (`/svn/Calculadora`)  
- [ ] Conexión desde Eclipse confirmada  

---

**Documento elaborado por:**  
*Nicolás Ochaita y equipo de Ingeniería de Software 2 – UMES 2025*
