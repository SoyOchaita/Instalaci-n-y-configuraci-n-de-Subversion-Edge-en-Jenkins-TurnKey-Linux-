# 🧩 Proyecto Ingeniería de Software 2 – Fase 1  
## Instalación y configuración de Subversion Edge en Jenkins (TurnKey Linux) – Modo Offline (con Webmin)

**Autor:** Alfonso Ochaita– Universidad Mesoamericana  
**Curso:** Ingeniería de Software 2 (2025)  
**Fase:** 1 – Instalación y configuración del sistema de control de versiones SVN  
**Créditos:** Imagen Docker [`svnedge/app`](https://hub.docker.com/r/svnedge/app/tags) publicada por *svnedge* en Docker Hub.  
Todos los derechos pertenecen a sus autores originales.  

---

## 🎯 Objetivo

Instalar y configurar **Subversion Edge (SVN Edge)** dentro de una máquina virtual **TurnKey Jenkins (Debian 12 “Bookworm”)** sin conexión a Internet, utilizando **Webmin** para administrar archivos, ejecutar comandos y transferir paquetes offline.  
El resultado es un entorno completamente funcional de **control de versiones** vinculado con Eclipse mediante el plugin **Subclipse**.

---

## ⚙️ Archivos necesarios

Antes de iniciar, asegúrate de tener en tu equipo local los siguientes archivos comprimidos:

```
docker-offline-packages.tar.gz   ← contiene los 5 paquetes .deb de Docker
svnedge-app.tar.gz               ← imagen Docker de Subversion Edge
```

Ambos archivos deben ser **subidos manualmente a la máquina Jenkins** mediante **Webmin**.

---

## 🪄 PASOS COMPLETOS

### 1️⃣ Subir los archivos a Jenkins mediante Webmin

1. Abre tu navegador y entra al panel de Webmin:  
   ```
   https://IP_DE_TU_VM:1231 (o el que indique el JenkinsServer)
   ```
2. Ve a: **Others → File Manager**
3. Navega hasta la ruta `/root/`
4. Usa el botón **Upload to current directory**
5. Sube los dos archivos:
   - `docker-offline-packages.tar.gz`
   - `svnedge-app.tar.gz`

Verifica que queden en `/root/`.

---

### 2️⃣ Abrir la consola en Webmin

1. En el menú lateral, entra en:  
   **Others → Command Shell** o **Others → Terminal**
2. Ya estás en la consola como **root** (no es necesario usar `sudo`).

---

### 3️⃣ Descomprimir los archivos subidos

```bash
cd /root
tar -xzvf docker-offline-packages.tar.gz
gunzip svnedge-app.tar.gz
```

Esto genera las siguientes estructuras:

```
/root/docker-offline/      ← contiene los 5 archivos .deb
/root/svnedge-app.tar      ← imagen de Subversion Edge lista para importar
```

---

### 4️⃣ Instalar Docker desde los archivos .deb

```bash
cd /root/docker-offline
dpkg -i *.deb
systemctl enable docker
systemctl start docker
docker --version
```

✅ Si el comando `docker --version` muestra una versión (por ejemplo `Docker version 28.5.1`), la instalación fue correcta.

---

### 5️⃣ Cargar la imagen `svnedge/app` en Docker

```bash
cd /root
docker load -i svnedge-app.tar
```

Confirma que la imagen fue importada:
```bash
docker images
```
Debería aparecer:
```
REPOSITORY     TAG       IMAGE ID       SIZE
svnedge/app    latest    9e077ed8a6df   590MB
```

---

### 6️⃣ Crear y ejecutar el contenedor

Si existía un contenedor anterior, elimínalo:
```bash
docker rm -f svnedge
```

Crea el contenedor con los puertos requeridos:
```bash
docker run -d   -p 3344:3343   -p 18180:18080   --restart always   --name svnedge   svnedge/app
```

✅ Verifica que esté corriendo:
```bash
docker ps
```
Deberías ver:
```
PORTS
0.0.0.0:3344->3343/tcp, 0.0.0.0:18180->18080/tcp
```

📘 **Puertos utilizados:**
| Interno | Externo | Función |
|----------|----------|----------|
| 3343 | 3344 | Servicio Subversion (repositorios) |
| 18080 | 18180 | Panel web y navegador ViewVC |

---

### 7️⃣ Acceder a la interfaz web

Desde tu computadora local (fuera de la VM), abre en tu navegador:

- **Panel de administración:**  
  `http://IP_DE_TU_VM:18180`
- **Navegador de código (ViewVC):**  
  `http://IP_DE_TU_VM:18180/viewvc`
- **Repositorio directo (para Eclipse/Subclipse):**  
  `http://IP_DE_TU_VM:3344/svn/Calculadora`

> 📌 Reemplaza `IP_DE_TU_VM` por la IP real de tu máquina Jenkins (puedes verla con `ip a` o en Webmin → Network).

---

### 8️⃣ Configurar Subversion Edge

1. Abre el panel principal (`http://IP_DE_TU_VM:18180`)
2. Crea el usuario **admin** y completa el asistente de configuración
3. Acepta los términos de licencia
4. En la pestaña **Administration → Server**, presiona **Start SVN Server**
5. En el menú lateral → **Repositories → Create**
6. Nombre del repositorio:  
   ```
   Calculadora
   ```
7. Guarda los cambios → debe aparecer con estado **OK**

---

### 9️⃣ Verificar repositorio desde terminal

```bash
svn list http://IP_DE_TU_VM:3344/svn/Calculadora --username admin
```
Debe pedir contraseña y mostrar un listado vacío (repositorio nuevo y funcional).

---

### 🔟 Conectar desde Eclipse (Subclipse)

1. Instalar el plugin **Subclipse** desde el Marketplace  
2. Abrir la perspectiva **SVN Repository Exploring**  
3. Crear una nueva conexión con:  
   ```
   URL: http://IP_DE_TU_VM:3344/svn/Calculadora
   Usuario: admin
   Contraseña: (la definida en el panel)
   ```
4. Crear el proyecto Java **Calculadora**
5. Estructura mínima solicitada:

```
Calculadora/
 ├── src/gt/edu/umes/proy/
 │    ├── Calculadora.java        # contiene main()
 │    └── LogicaDelNegocio.java   # funciones suma y promedio
 ├── .classpath
 ├── .project
 └── README.md
```
6. Compartir proyecto con SVN → **Team → Share Project → SVN → Commit inicial**

---

## 🧰 Solución de problemas comunes

| Problema | Causa | Solución |
|-----------|--------|-----------|
| `404 /svn/Calculadora` | Falta exponer puertos | Recrear contenedor con `-p 3344:3343 -p 18180:18080` |
| `DNS_PROBE_FINISHED_NXDOMAIN` | Se intenta acceder con el nombre del contenedor | Usar la IP real de la VM Jenkins |
| `address already in use` | Puerto 18080 ya usado por Jenkins | Cambiar puerto externo (`-p 18181:18080`) |
| No inicia el panel | Docker detenido | `systemctl start docker` y `docker ps` |
| Repos desaparecen | Falta volumen persistente | Agregar `-v /var/svn-data:/var/opt/csvn` al comando `docker run` |

---

## 💾 Cómo descargar archivos desde Webmin a tu computadora local

1. Entra a Webmin:  
   `https://IP_DE_TU_VM:12321 (o el que indique JenkinsServer) `
2. Ve a: **Others → File Manager**
3. Navega a `/root/`
4. Selecciona el archivo (`docker-offline-packages.tar.gz` o `svnedge-app.tar.gz`)
5. Haz clic en el botón **Download** ⬇️  
   → El archivo se descargará directamente a tu computadora 💻

---

## 📜 Créditos y licencia

- **Imagen Docker:** [`svnedge/app`](https://hub.docker.com/r/svnedge/app/tags) por *svnedge* (Docker Hub)  
- **Subversion Edge:** desarrollado por *CollabNet, Inc.*  
- Documento creado con fines académicos en la **Universidad Mesoamericana (UMES 2025)**

---

## ✅ Verificación final

| Paso | Resultado esperado |
|------|---------------------|
| Docker instalado (offline) | ✅ `docker --version` visible |
| Imagen `svnedge/app` cargada | ✅ aparece en `docker images` |
| Contenedor activo | ✅ visible en `docker ps` |
| Panel accesible | ✅ `http://IP_VM:18180` muestra la interfaz |
| ViewVC disponible | ✅ `http://IP_VM:18180/viewvc` abre navegador |
| Repositorio creado | ✅ `Calculadora` visible en panel |
| Conexión desde Eclipse | ✅ Checkout y commit exitosos |

---

**Documento actualizado:** Octubre 2025  
**Elaborado por:** *Nicolás Ochaita y equipo – Ingeniería de Software 2, UMES 2025*
