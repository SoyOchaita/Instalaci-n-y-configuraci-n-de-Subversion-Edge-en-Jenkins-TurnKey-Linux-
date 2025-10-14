[README_SVNEDGE_OFFLINE_v2.md](https://github.com/user-attachments/files/22915264/README_SVNEDGE_OFFLINE_v2.md)
# Proyecto Ingeniería de Software 2 – Fase 1  
## Jenkins (TurnKey Linux) + Subversion Edge **offline** con Docker

**Autor:** Equipo de Ingeniería de Software 2 (UMES 2025)  
**Mantenedor guía:** Nicolás Ochaita (líder de integración)  
**Créditos:** Imagen Docker **`svnedge/app`** publicada por el autor *svnedge* en Docker Hub (ver tags y detalles en su página). Sin esa imagen pública este procedimiento no sería posible. Respeta su licencia y términos de uso.

---

## Objetivo

Levantar **Subversion Edge (SVN Edge)** dentro de una **VM Jenkins TurnKey (Debian 12 “bookworm”)** sin acceso a Internet, **usando Docker en modo offline**, crear el repositorio **`Calculadora`** y dejar URLs de acceso para Eclipse (Subclipse) y el navegador (ViewVC).

---

## Visión general del flujo

1) En una máquina **con Internet** (WSL/Ubuntu o cualquier Debian/Ubuntu):  
   - Agregar el repo oficial de **Docker para Debian**.  
   - Descargar los **paquetes .deb** de Docker **sin instalarlos** (`apt download`).  
   - (Opcional) Exportar la imagen `svnedge/app` a un `.tar` con `docker save`.  
   - Calcular checksums y **comprimir** todo para traslado.

2) En la **VM Jenkins (sin Internet)**:  
   - Subir los `.deb` y el `.tar` por **Webmin / SCP**.  
   - Instalar Docker con `dpkg -i *.deb`.  
   - Cargar la imagen `svnedge/app` con `docker load`.  
   - Ejecutar el contenedor publicando puertos **3344** (SVN) y **18180** (UI/ViewVC).  
   - Crear el repositorio **`Calculadora`** y probar acceso.

---

## Requisitos

- VM **TurnKey Jenkins** basada en Debian 12 (bookworm).  
- Acceso **root** a la VM.  
- **Webmin** activo o SSH para subir archivos.  
- Un equipo auxiliar **con Internet** (puede ser tu WSL/Ubuntu).

> Comprueba la versión de tu VM: `cat /etc/os-release` → debe decir `Debian GNU/Linux 12 (bookworm)`.

---

## 1) Preparación **en una máquina con Internet**

> Ejemplos con **Ubuntu/WSL** (sirve también en Debian).

### 1.1 Agregar el repositorio oficial de Docker **para Debian**
> *OJO:* aunque estés en Ubuntu/WSL, si vas a descargar paquetes para una VM Debian 12, **haz estos pasos en una máquina Debian 12 con Internet**, o usa el mismo procedimiento pero apuntando a Debian (no a Ubuntu). La forma más simple: usa un **contenedor Debian:bookworm** o una VM Debian con Internet para “bajar” los `.deb`.

```bash
sudo apt update
sudo apt install -y ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
https://download.docker.com/linux/debian \
$(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update
```

### 1.2 Descargar **sin instalar** los paquetes `.deb`
```bash
mkdir -p ~/docker-offline
cd ~/docker-offline

apt download \
  containerd.io \
  docker-ce \
  docker-ce-cli \
  docker-buildx-plugin \
  docker-compose-plugin
```

Archivos esperados (nombres pueden variar por versión):
```
containerd.io_1.7.xx-1~debian.12~bookworm_amd64.deb
docker-ce_5:28.x.x-1~debian.12~bookworm_amd64.deb
docker-ce-cli_5:28.x.x-1~debian.12~bookworm_amd64.deb
docker-buildx-plugin_0.29.x-1~debian.12~bookworm_amd64.deb
docker-compose-plugin_2.40.x-1~debian.12~bookworm_amd64.deb
```

### 1.3 (Opcional pero recomendado) Exportar la imagen `svnedge/app`
> **Créditos**: Imagen publicada por *svnedge* en Docker Hub. Consulta sus **tags** y términos en su página oficial.
```bash
docker pull svnedge/app:latest
docker save -o svnedge-app.tar svnedge/app:latest
gzip -9 svnedge-app.tar   # genera svnedge-app.tar.gz
```

### 1.4 Verificación e integridad
```bash
sha256sum *.deb svnedge-app.tar.gz > SHA256SUMS.txt
```

### 1.5 Empaquetar para traslado
```bash
cd ..
tar -czvf svnedge_offline_bundle.tar.gz docker-offline/ SHA256SUMS.txt
```
El archivo `svnedge_offline_bundle.tar.gz` contiene **todos los .deb + checksums**. Lleva también `svnedge-app.tar.gz` si lo generaste.

---

## 2) Instalación **en la VM Jenkins (sin Internet)**

### 2.1 Subir archivos a la VM
Métodos sugeridos:
- **Webmin → Others → File Manager → Upload**  
- **SCP** desde tu PC con Internet.

Ruta sugerida en la VM:
```
/root/docker-offline/   # .deb
/root/svnedge-app.tar.gz
/root/SHA256SUMS.txt    # si lo generaste
```

### 2.2 (Opcional) Verificar integridad
```bash
cd /root
sha256sum -c SHA256SUMS.txt
# Debe decir "OK" para cada archivo
```

### 2.3 Instalar Docker **offline**
```bash
cd /root/docker-offline
dpkg -i *.deb
systemctl enable docker
systemctl start docker
docker --version
```

### 2.4 Cargar la imagen `svnedge/app`
```bash
cd /root
gunzip -t svnedge-app.tar.gz  # prueba compresión
gunzip svnedge-app.tar.gz
docker load -i svnedge-app.tar
docker images | grep svnedge
```

Salida esperada:
```
svnedge/app   latest   <IMAGE_ID>   590MB (aprox.)
```

### 2.5 Ejecutar el contenedor publicando puertos
> Si el puerto 18080 del host ya está en uso por Jenkins/Tomcat, expón **18180** hacia el **18080** interno.

```bash
docker rm -f svnedge 2>/dev/null || true

docker run -d \
  -p 3344:3343 \
  -p 18180:18080 \
  --restart always \
  --name svnedge \
  svnedge/app
```

Verifica:
```bash
docker ps
# Esperado: 0.0.0.0:3344->3343/tcp, 0.0.0.0:18180->18080/tcp
```

> **3344** (host) ↔ **3343** (contenedor) = servicio SVN  
> **18180** (host) ↔ **18080** (contenedor) = panel Web + ViewVC

---

## 3) Configuración de Subversion Edge

### 3.1 Abrir interfaz web
Navegador → `http://IP_DE_LA_VM:18180`  
Crea el usuario **admin** y completa el asistente.

### 3.2 Confirmar estado del servidor SVN
En el dashboard debe aparecer **Subversion status: Up**. Si no, **Start**.

### 3.3 Crear el repositorio del proyecto
- Menú **Repositories → Create**
- Nombre: **`Calculadora`**
- Guardar.

### 3.4 Probar URLs de acceso
- **Panel (administración):** `http://IP_VM:18180`  
- **ViewVC (browser de código):** `http://IP_VM:18180/viewvc`  
- **Repositorio (SVN, para Eclipse/Subclipse):**  
  `http://IP_VM:3344/svn/Calculadora`

> Reemplaza `IP_VM` por la IP real de tu Jenkins (ej. `192.168.1.36`).

---

## 4) Conectar desde Eclipse (Subclipse)

1. Instalar **Subclipse** desde Marketplace.  
2. `SVN Repository Exploring` → **New Repository Location** → URL:  
   `http://IP_VM:3344/svn/Calculadora`  
3. Usuario/clave (el que creaste en SVN Edge).  
4. Crea el proyecto **Calculadora** (Java SE).  
5. Estructura mínima solicitada por el curso:

```
Calculadora/
 ├── src/gt/edu/umes/proy/
 │    ├── Calculadora.java        # contiene main(String[] args)
 │    └── LogicaDelNegocio.java   # validaciones + suma + promedio
 ├── .classpath
 ├── .project
 └── README.md
```

6. `Team → Share Project → SVN` → selecciona el repo → **Commit** inicial.

---

## 5) Solución de problemas comunes

- **404 en `/svn/Calculadora` o `/viewvc`**  
  → Asegúrate de publicar **ambos puertos**: `3344:3343` y `18180:18080`.  
  → En el navegador usa **18180** para el panel/ViewVC; **3344** solo para SVN.

- **DNS_PROBE_FINISHED_NXDOMAIN** con `http://<ID_DEL_CONTENEDOR>:`  
  → Usa la **IP de la VM**, no el nombre interno del contenedor.

- **“address already in use” al crear el contenedor**  
  → Cambia el puerto externo (ej. `-p 18181:18080`) o libera el puerto.

- **No veo el panel pero sí el contenedor**  
  → `docker logs -f svnedge` y revisa errores.  
  → `systemctl status docker` (que Docker esté activo).

- **Persistencia de datos** (recomendado):  
  Recrea el contenedor montando un volumen:
  ```bash
  docker rm -f svnedge
  mkdir -p /var/svn-data
  docker run -d \
    -p 3344:3343 -p 18180:18080 \
    -v /var/svn-data:/var/opt/csvn \
    --restart always \
    --name svnedge \
    svnedge/app
  ```
  Ahora los repos viven en `/var/svn-data` del host.

---

## Créditos y licencias

- **Imagen Docker `svnedge/app`** → Autor: *svnedge* (Docker Hub).  
  Atribución: “SVN Edge Official Release Image”. Respeta los **términos/licencias** de la imagen y del software Subversion Edge.  
- Documentación de **Docker Engine para Debian**: repos oficiales de Docker.  
- Esta guía describe un procedimiento **educativo** para fines académicos (UMES 2025).

---

## Checklist de verificación rápida (para docencia)

- [ ] Se instaló Docker **offline** desde `.deb`  
- [ ] Se cargó la imagen `svnedge/app` con `docker load`  
- [ ] Contenedor corre con puertos `3344` y `18180` publicados  
- [ ] Accede a **panel**: `http://IP_VM:18180`  
- [ ] Accede a **ViewVC**: `http://IP_VM:18180/viewvc`  
- [ ] Accede a **repo**: `http://IP_VM:3344/svn/Calculadora`  
- [ ] Eclipse (Subclipse) hace **checkout/commit** correctamente  
