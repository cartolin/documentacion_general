<!-- docs/docker/concept.md -->

# 🔍 Conceptos básicos de Docker

Docker es una plataforma de **contenedores** que permite:

1. **Empaquetar** una aplicación junto con todas sus dependencias en una **imagen** ligera.  
2. **Ejecutar** esa imagen en cualquier servidor o equipo local, garantizando que el entorno sea idéntico.  
3. **Aislar** servicios: cada contenedor corre separado, sin interferir con otros.  
4. **Versionar** y **compartir** imágenes a través de registros públicos o privados (Docker Hub, GitHub Container Registry, etc.).

---

## Componentes clave

- **Docker Engine**: demonio que construye y corre contenedores.  
- **Docker CLI**: herramienta de línea de comandos (`docker build`, `docker run`, …).  
- **Imágenes**: “plantillas” inmutables que describen un contenedor.  
- **Contenedores**: instancias en ejecución de una imagen.  
- **Dockerfile**: archivo de texto donde defines cómo construir tu imagen.

---

## Flujo típico

1. Escribes un **Dockerfile** con instrucciones (`FROM`, `COPY`, `RUN`, `CMD`).  
2. Ejecutas `docker build -t mi-app:latest .` → genera una imagen.  
3. Publicas la imagen en un registro (`docker push`).  
4. Levantas un contenedor en producción:  
   ```bash
   docker run -d --name mia plicación -p 80:3000 mi-app:latest
