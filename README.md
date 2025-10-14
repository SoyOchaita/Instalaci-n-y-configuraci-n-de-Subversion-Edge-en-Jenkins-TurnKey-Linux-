[README_SVNEDGE_OFFLINE.md](https://github.com/user-attachments/files/22915175/README_SVNEDGE_OFFLINE.md)
# 🧩 Proyecto Ingeniería de Software 2 – Fase 1  
## Instalación y configuración de Subversion Edge en Jenkins (TurnKey Linux)

**Autor:** Grupo de Ingeniería de Software 2  
**Curso:** UMES – 2025  
**Fase:** 1 – Instalación y Configuración del Sistema de Control de Versiones  
**Profesor:** _(agregar nombre del catedrático)_

---

## 🎯 Objetivo

Instalar y configurar **Subversion Edge (SVN Edge)** dentro de una **máquina virtual Jenkins (TurnKey)** utilizando **Docker offline**, para poder crear el repositorio del proyecto **“Calculadora”** y conectarlo desde Eclipse mediante el plugin **Subclipse**.

---

## ⚙️ Requisitos previos

- Máquina virtual con **TurnKey Jenkins (Debian 12 Bookworm)**  
- Acceso a **Webmin** o **terminal root**
- Archivo `svnedge-app.tar.gz` exportado desde Docker en WSL
- Paquetes `.deb` de Docker descargados previamente (modo offline)

---

## 🧱 Estructura de archivos antes de iniciar

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

---

## 🪄 PASOS DETALLADOS

### 1️⃣ Descomprimir la imagen de Subversion Edge

```bash
cd /root
gunzip svnedge-app.tar.gz
```

Resultado:  
```
svnedge-app.tar
```

---

### 2️⃣ Instalar Docker offline en la VM Jenkins

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

Debería mostrar algo como:
```
Docker version 28.5.1, build xxxxxxx
```

---

### 3️⃣ Cargar la imagen `svnedge/app` en Docker

```bash
docker load -i /root/svnedge-app.tar
```

Confirmar que la imagen esté disponible:

```bash
docker images
```

Resultado esperado:
```
REPOSITORY     TAG       IMAGE ID       SIZE
svnedge/app    latest    9e077ed8a6df   590MB
```

---

### 4️⃣ Crear el contenedor de Subversion Edge

Primero elimina contenedores anteriores si existían:

```bash
docker rm -f svnedge
```

Luego crea uno nuevo con los puertos correctamente mapeados:

```bash
docker run -d   -p 3344:3343   -p 18180:18080   --restart always   --name svnedge   svnedge/app
```

📍 **Explicación de puertos:**

| Puerto interno | Puerto externo | Descripción |
|-----------------|----------------|--------------|
| `3343` | `3344` | Servicio principal de Subversion (repositorios) |
| `18080` | `18180` | Interfaz web y ViewVC (panel de administración) |

Verifica que esté corriendo:

```bash
docker ps
```

Resultado esperado:
```
PORTS
0.0.0.0:3344->3343/tcp, 0.0.0.0:18180->18080/tcp
```

---

### 5️⃣ Acceder a la interfaz web de Subversion Edge

Desde un navegador en tu computadora:

- Panel de administración:  
  ```
  http://192.168.1.36:18180
  ```
- Repositorio directo (para Eclipse):  
  ```
  http://192.168.1.36:3344/svn/Calculadora
  ```
- Navegador de código (ViewVC):  
  ```
  http://192.168.1.36:18180/viewvc
  ```

> ⚠️ Reemplaza `192.168.1.36` con la IP real de tu máquina Jenkins.

---

### 6️⃣ Crear el repositorio del proyecto “Calculadora”

1. Inicia sesión en `http://192.168.1.36:18180`
2. Ve a **Repositories → Create**
3. Nombre del repositorio: `Calculadora`
4. Da clic en **Create**
5. Asigna permisos al usuario `admin` o `nicolas` con acceso **Read/Write**
6. Copia el **URL del repositorio:**
   ```
   http://192.168.1.36:3344/svn/Calculadora
   ```

---

### 7️⃣ Probar la conexión con Subversion

Desde terminal:
```bash
svn list http://192.168.1.36:3344/svn/Calculadora --username admin
```

Debe pedir contraseña y luego mostrar una lista vacía (repositorio nuevo).

---

### 8️⃣ Conectar desde Eclipse (Subclipse)

1. En Eclipse, instala el plugin **Subclipse** desde el Marketplace.  
2. Abre la perspectiva **SVN Repository Exploring**.  
3. Crea una nueva ubicación:
   ```
   URL: http://192.168.1.36:3344/svn/Calculadora
   Usuario: admin
   Contraseña: ****
   ```
4. Crea tu proyecto Java **Calculadora** y compártelo con Subversion (**Team → Share Project → SVN**).

---

### 9️⃣ Estructura de proyecto requerida

```
Calculadora/
 ├── src/gt/edu/umes/proy/
 │    ├── Calculadora.java
 │    └── LogicaDelNegocio.java
 ├── .classpath
 ├── .project
 └── README.md
```

---

## 🧠 Resumen general

| Etapa | Acción | Resultado |
|--------|--------|------------|
| 1 | Descomprimir imagen `svnedge-app.tar.gz` | Archivo de imagen Docker listo |
| 2 | Instalar Docker offline | Docker operativo sin internet |
| 3 | Cargar imagen SVN Edge | Imagen `svnedge/app` disponible |
| 4 | Ejecutar contenedor con puertos `3344` y `18180` | SVN Edge activo y accesible |
| 5 | Crear repositorio `Calculadora` | Repositorio disponible en URL |
| 6 | Conectar desde Eclipse (Subclipse) | Proyecto compartido en SVN |

---

## 🔒 Notas finales

- Si reinicias la VM, el contenedor se iniciará automáticamente gracias a `--restart always`.  
- Puedes ver el estado del contenedor en cualquier momento con:
  ```bash
  docker ps
  ```
- Para detenerlo:
  ```bash
  docker stop svnedge
  ```
- Para iniciarlo nuevamente:
  ```bash
  docker start svnedge
  ```
