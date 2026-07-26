# Guía de examen — Docker / Kubernetes / CI-CD

Guía rápida de escanear, no de leer. Un flujo numerado por reto.

---

## Preparación (una sola vez al inicio)

```cmd
cd <carpeta_de_la_app>
git init
git add .
git commit -m "estado inicial: app sin dockerizar"
git branch -M main
git remote add origin <URL_del_repo>.git
git push -u origin main
```

Después de cada reto resuelto, siempre:
```cmd
git add .
git commit -m "fix: <qué corregiste>"
git push
```

---

## Glosario rápido
- **Imagen** = receta empaquetada (no corre). **Contenedor** = la receta corriendo.
- **`EXPOSE`** solo documenta — no cambia en qué puerto escucha el proceso de verdad. Eso lo define el código (`app.listen(PORT)`).
- **Pod** = un contenedor corriendo dentro de Kubernetes.
- **Service** = dirección fija que reparte tráfico entre Pods vivos.
- **Manifiesto (.yaml)** = describe el estado deseado; Kubernetes decide cómo lograrlo.

---

## Reto 1 — El contenedor "corre" pero no responde

**El bug:** `docker ps` muestra `Up`, pero no hay respuesta. Vivo ≠ escuchando en el puerto correcto.

### Secuencia (el Dockerfile con el bug ya lo da el profe)

1. **Revisar el código real** (ej. `server.js`) → confirmar en qué puerto escucha de verdad (`process.env.PORT || 3000` → 3000).
2. **Comparar contra el Dockerfile dado** → ver si `EXPOSE` coincide con ese puerto.
3. **Construir la imagen** (con el bug, para reproducirlo):
   ```cmd
   docker build -t <nombre_imagen> .
   ```
4. **Correr el contenedor** (`--name` = tú se lo pones, si no Docker le da uno random):
   ```cmd
   docker run -d -p <puerto>:<puerto> --name <nombre_contenedor> <nombre_imagen>
   ```
5. **Confirmar el síntoma:**
   ```cmd
   docker ps
   curl http://localhost:<puerto>/health
   ```
   → falla o se cuelga. Bug reproducido.
6. **Diagnosticar:**
   ```cmd
   docker logs <nombre_contenedor>
   ```
   → revela el puerto real (mensaje de consola de la app). Si hace falta más detalle:
   ```cmd
   docker exec -it <nombre_contenedor> sh
   netstat -tlnp
   ```
7. **Detener y borrar el contenedor viejo** (obligatorio antes de reconstruir):
   ```cmd
   docker stop <nombre_contenedor>
   docker rm <nombre_contenedor>
   ```
8. **Corregir el Dockerfile:** `EXPOSE <puerto_real>`
9. **Reconstruir la imagen** (siempre que cambias el Dockerfile, hay que repetir el build — la imagen vieja no se actualiza sola):
   ```cmd
   docker build -t <nombre_imagen> .
   ```
10. **Correr el contenedor corregido** (¡el `-p` también debe usar el puerto real!):
    ```cmd
    docker run -d -p <puerto_real>:<puerto_real> --name <nombre_contenedor> <nombre_imagen>
    ```
11. **Verificar:**
    ```cmd
    curl http://localhost:<puerto_real>/health
    ```
    → `{"status":"ok"}` = resuelto.

### Si el examen trae otra variante del mismo bug
- App escucha en `127.0.0.1` en vez de `0.0.0.0` → se arregla en el código, no en el Dockerfile.
- `CMD` corre el archivo equivocado o falla una instalación silenciosamente.

El método no cambia: `docker logs` primero, `docker exec` si necesitas más detalle.

### ✅ Checklist
- [x] Reproduje el bug
- [x] Diagnostiqué con logs/exec
- [x] Corregí Dockerfile + mapeo de puertos
- [x] Verifiqué desde fuera
- [x] Commit antes/después

---

*(Siguiente: Reto 2 — Pods en Running, Service sin tráfico)*
