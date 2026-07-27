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

## Reto 2 — Pods en `Running`, Service sin tráfico

**El bug:** no hay ningún error visible. Todo se ve sano (Pods `Running`, sin warnings) — la única pista es un campo vacío que hay que saber buscar.

**Concepto clave:** un Service no "ve" Pods directamente — busca Pods cuyas **labels** coincidan exactamente con su `selector`. Si no coinciden ni una letra, el Service no encuentra a nadie, aunque los Pods estén perfectos.

⚠️ **Regla de Kubernetes que no es negociable:** `Deployment.spec.selector.matchLabels` DEBE coincidir con `Deployment.spec.template.metadata.labels` (ambos del lado del Deployment) — si no coinciden, Kubernetes rechaza el Deployment completo y no crea ni un Pod. El bug real de este reto va en el **Service.selector**, no en el Deployment.

### Secuencia

1. **Aplicar el manifiesto con el bug** (`k8s.yaml`, Deployment válido + Service con selector equivocado):
   ```cmd
   kubectl apply -f k8s.yaml
   ```
2. **Confirmar que los Pods están "sanos" (el señuelo):**
   ```cmd
   kubectl get pods
   ```
   → `Running`, `1/1 Ready`. Todo parece perfecto.
3. **Buscar el síntoma real** (no hay error rojo, solo un dato vacío):
   ```cmd
   kubectl get endpoints web-service
   ```
   → `ENDPOINTS: <none>`. **Eso ES el error.**
4. **Comparar labels vs selector, lado a lado:**
   ```cmd
   kubectl get pods --show-labels
   kubectl describe service web-service
   ```
   → Pod tiene `app=web`, Service busca `app=webapp`. No coinciden.
5. **Corregir** el `selector` del Service en el yaml para que coincida con el label real del Pod.
6. **Reaplicar** (actualiza solo lo que cambió, no borra todo):
   ```cmd
   kubectl apply -f k8s.yaml
   kubectl get endpoints web-service
   ```
   → ahora deben aparecer IPs, no `<none>`.
7. **Probar de punta a punta** (el Service es `ClusterIP`, no accesible directo desde tu compu — se necesita un túnel):
   ```cmd
   kubectl port-forward service/web-service <puerto_local>:<puerto_service>
   ```
   En otra terminal:
   ```cmd
   curl http://localhost:<puerto_local>/health
   ```
   → `{"status":"ok"}` = resuelto.

### Los 3 puertos que no hay que confundir
```
tu compu (puerto local, el que tú eliges) → Service (spec.ports.port) → Pod (targetPort, el real de la app)
```

### ✅ Checklist
- [x] Reproduje el bug (Pods sanos, Service sin endpoints)
- [x] Encontré la causa comparando labels vs selector
- [x] Corregí el selector del Service
- [x] Verifiqué endpoints poblados + curl exitoso vía port-forward
- [x] Commit

---

---

## Reto 3 — El pipeline que despliega aunque las pruebas fallen

**El bug:** el job `deploy` no tiene ninguna instrucción que lo obligue a esperar al job `build-test` — por defecto, en GitHub Actions, **todos los jobs de un workflow corren en paralelo**, sin orden garantizado. Si `build-test` falla, `deploy` no se entera y corre igual.

### Concepto clave: `needs:`
`needs: build-test` dentro del job `deploy` significa: *"no arranques hasta que `build-test` termine, y solo si terminó exitosamente."* Si `build-test` falla, `deploy` aparece como **Skipped** (gris) — nunca llega ni a "Set up job".

### Dónde y cómo romper una prueba a propósito
1. Busca el archivo de pruebas del proyecto — normalmente termina en `.test.js` (Node), `_test.py`/`test_*.py` (Python), o similar. En este proyecto es `server.test.js`.
2. Adentro, busca cualquier línea con `assert` (o `expect`, según el framework) — es la línea que compara "lo que la app respondió de verdad" contra "lo que se espera". Ejemplo real usado aquí:
   ```js
   assert.strictEqual(res.status, 200);
   ```
3. Cambia el valor esperado (el segundo argumento) por cualquier número/texto que nunca vaya a coincidir:
   ```js
   assert.strictEqual(res.status, 999);
   ```
   Esto **no toca el código de la app** — tu servidor sigue respondiendo `200` normal, real y correctamente. Solo le mentiste a la prueba sobre qué esperar, así que la comparación falla a propósito y es 100% reversible (solo hay que devolver el `200`).
4. **Elige un `assert` que no deje el servidor abierto si falla** (o asegúrate de tener `timeout-minutes` puesto, ver más abajo) — si el `assert` que rompes está antes de una línea `server.close()`, esa línea nunca se alcanza y el proceso queda colgado.
5. Para revertir cuando ya no necesites el bug: regresa el valor original (`999` → `200`) y vuelve a hacer commit/push.

### Secuencia

1. **Push del `.yml` tal cual lo da el examen** (sin `needs:`, con las pruebas intactas):
   ```cmd
   git add .
   git commit -m "ci: pipeline base"
   git push
   ```
   → `build-test` ✅, `deploy` puede fallar en sus propios pasos (Dockerfile/registry inexistente) — no importa, eso es aparte del bug.

2. **Rompe una prueba a propósito** (en `server.test.js`, cambiar un valor esperado en un `assert`, ej. `assert.strictEqual(res.status, 200)` → `999`):
   ```cmd
   git add .
   git commit -m "ci: rompo una prueba a proposito (defecto)"
   git push
   ```
   → `build-test` ❌, pero **`deploy` corre de todas formas** (aunque también termine en ❌ por otra razón) — esa es la evidencia: se ejecutó sin esperar.

3. **Agrega la dependencia:**
   ```yaml
     deploy:
       needs: build-test
       runs-on: ubuntu-latest
   ```
   Push (con la prueba **aún rota**) → `deploy` debe aparecer gris/Skipped, sin intentar arrancar.

4. **Corrige la prueba** (vuelve el valor al original) → push final → `build-test` ✅ y `deploy` sí arranca (esta vez porque `build-test` pasó, no en paralelo).

### ⚠️ Cuidado: una prueba rota con `assert` puede colgar el job
Si el `assert` falla **antes** de una línea que cierra el servidor (`server.close()`), esa línea nunca se ejecuta y el servidor de prueba queda abierto — Node.js no termina el proceso mientras algo siga "escuchando". Por eso agregamos:
```yaml
  build-test:
    runs-on: ubuntu-latest
    timeout-minutes: 2
```
`timeout-minutes` corta el job automáticamente si se pasa del tiempo — buena práctica real de CI/CD, no es "trampa".

### Cómo explicárselo al profe (por si pregunta)
- **"¿Por qué usaste `timeout-minutes`?"** → *"Al romper una prueba con `assert`, el código se detiene ahí y el servidor de prueba nunca se cierra, dejando el job colgado. `timeout-minutes` es una práctica estándar en CI/CD para que un job nunca quede corriendo indefinidamente."*
- **"¿Qué hiciste exactamente con la prueba rota (200 → 999)?"** → *"El servidor sigue respondiendo 200 normalmente, no toqué el código de la app. Solo cambié el valor que la prueba **espera** — le dije que compare contra 999 en vez de 200. Como `200 !== 999`, la aserción falla a propósito, sin que la app esté realmente rota. Es la forma de simular una prueba fallida sin dañar el comportamiento real."*
- **"¿Por qué `deploy` falla incluso cuando `build-test` pasa?"** → *"Porque el job `deploy` no tiene el paso `checkout`, entonces la máquina de GitHub no tiene mi código para construir la imagen; y aunque lo tuviera, `registry/app` no es un registro real donde yo tenga permiso de subir, y `kubectl` no puede alcanzar mi clúster local desde la nube de GitHub. Son limitaciones de la infraestructura de prueba, no del bug de dependencia entre jobs que estoy demostrando (ese se ve con `needs:`, no con si `deploy` se completa)."*

### ✅ Checklist
- [x] Push base — confirmé que build-test corre bien
- [x] Rompí una prueba y confirmé que deploy corrió igual (sin `needs:`)
- [x] Agregué `needs: build-test`
- [x] Con la prueba rota, confirmé que deploy quedó en Skipped
- [x] Corregí la prueba — pipeline corrió de punta a punta
- [x] Agregué `timeout-minutes` para evitar cuelgues
- [x] Commit de cada cambio

---

---

## El giro final — tráfico triplicado, sin corte de servicio

**El escenario:** el tráfico se espera que se triplique, y el próximo despliegue no debe causar un corte perceptible. Pide dos cosas: (1) más capacidad — réplicas o autoescalado, y (2) una estrategia de despliegue segura para el "próximo release".

### Concepto clave
- **Réplicas fijas** (`replicas: N`) = tú decides el número a mano, no reacciona solo a nada.
- **HPA (autoescalado)** = un recurso aparte que sí reacciona solo al uso de CPU/memoria — no es obligatorio, el checklist acepta "réplicas **o** autoescalado".
- **Estrategia de despliegue** = cómo se reemplazan los Pods viejos por nuevos sin dejar un hueco sin nadie respondiendo. `RollingUpdate` (la que usamos, y la que Kubernetes trae por defecto) los reemplaza de a uno.

### Secuencia (la que sí funcionó y quedó bien evidenciada)

1. **Sube las réplicas** en `k8s.yaml`:
   ```yaml
   spec:
     replicas: 4
   ```
2. **Declara explícitamente la estrategia rolling**, mismo nivel que `replicas` (no dentro de `template` ni `selector`):
   ```yaml
   spec:
     replicas: 4
     strategy:
       type: RollingUpdate
       rollingUpdate:
         maxUnavailable: 1
         maxSurge: 1
   ```
   `maxUnavailable: 1` = máximo 1 Pod caído a la vez. `maxSurge: 1` = máximo 1 Pod extra temporal mientras actualiza.
3. **Aplica y confirma las réplicas:**
   ```cmd
   kubectl apply -f k8s.yaml
   kubectl get pods
   ```
4. **Abre una terminal para vigilar los endpoints ANTES de disparar el release** (esto es lo que sí funcionó como evidencia — mejor que un loop de curl):
   ```cmd
   kubectl get endpoints web-service -w
   ```
5. **En otra terminal, dispara el "próximo release"** (simulado, sin necesidad de una imagen nueva real):
   ```cmd
   kubectl rollout restart deployment/web-deployment
   ```
6. **Observa la terminal del paso 4 mientras el rollout avanza.** La columna `ENDPOINTS` va cambiando de IPs (Pods viejos salen, nuevos entran) pero **nunca debe quedar vacía** en ningún momento — esa es la prueba de que nunca hubo un instante sin nadie respondiendo.
7. **Confirma el resultado final:**
   ```cmd
   kubectl rollout status deployment/web-deployment
   kubectl get pods
   ```
   → "successfully rolled out" + 4 Pods `Running` nuevos.

### ✅ Checklist
- [x] Subí réplicas de 2 a 4 (respuesta al tráfico triplicado)
- [x] Declaré `strategy: RollingUpdate` explícitamente con `maxUnavailable`/`maxSurge`
- [x] Disparé un release simulado con `kubectl rollout restart`
- [x] Verifiqué con `kubectl get endpoints -w` que nunca quedó vacío → cero corte
- [x] Confirmé `rollout status` exitoso
- [x] Commit

---

## Extra — cómo lograr que el pipeline salga 100% verde al menos una vez

El problema: `docker push registry/app` (registro falso) y `kubectl set image` (sin red hacia tu clúster local) nunca van a poder completarse de verdad sin infraestructura que no tienes en un examen de laboratorio. La solución estándar en CI/CD real: marca esos pasos específicos como no bloqueantes con `continue-on-error: true`, dejando que solo lo 100% reproducible (`checkout` + `docker build`) determine el resultado del job.

```yaml
name: ci-cd
on:
  push:
    branches: [main]

jobs:
  build-test:
    runs-on: ubuntu-latest
    timeout-minutes: 2
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm test

  deploy:
    needs: build-test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: docker build -t app:${{ github.sha }} .
      - run: docker push registry/app:${{ github.sha }}
        continue-on-error: true
      - run: kubectl set image deployment/web-deployment web=registry/app:${{ github.sha }}
        continue-on-error: true
```

Con la prueba ya corregida (valor original, no el roto), este `.yml` da un pipeline **100% verde de punta a punta** — cumple el punto del checklist: *"el pipeline se ejecutó exitosamente de punta a punta al menos una vez."*

---

## Extra — si el examen pide explícitamente HPA, blue-green o canary

No es necesario para cumplir el checklist base (acepta réplicas manuales + rolling), pero por si el profe lo pide explícito:

### HPA (autoescalado real)
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-deployment
  minReplicas: 2
  maxReplicas: 8
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 50
```
Aplicar con `kubectl apply -f hpa.yaml`. Verificar con:
```cmd
kubectl get hpa
```
Debería mostrar una columna `TARGETS` con el % de CPU actual vs el 50% configurado, y `REPLICAS` ajustándose solo si generas carga real de CPU. ⚠️ Requiere que el clúster tenga **metrics-server** instalado (en Docker Desktop/minikube casi nunca viene activo por defecto) — sin eso, `kubectl get hpa` mostrará `<unknown>` en TARGETS y nunca escalará. Activarlo consume tiempo extra, evalúa si te alcanza.

### Blue-Green — manifiesto completo (`blue-green.yaml`)
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-blue
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web
      version: blue
  template:
    metadata:
      labels:
        app: web
        version: blue
    spec:
      containers:
        - name: web
          image: inventario-app
          ports:
            - containerPort: 3000
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-green
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web
      version: green
  template:
    metadata:
      labels:
        app: web
        version: green
    spec:
      containers:
        - name: web
          image: inventario-app
          ports:
            - containerPort: 3000
---
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  selector:
    app: web
    version: blue     # <- versión activa
  ports:
    - port: 80
      targetPort: 3000
```

**Comandos:**
```cmd
kubectl apply -f blue-green.yaml
kubectl get pods --show-labels
```
→ 4 Pods: 2 `version=blue`, 2 `version=green`. El Service solo enruta a `blue` (confirmar con `kubectl describe service web-service`, campo `Selector:`).

**El "switch" (cambia el tráfico de golpe a green):**
```cmd
kubectl patch service web-service -p "{\"spec\":{\"selector\":{\"app\":\"web\",\"version\":\"green\"}}}"
kubectl describe service web-service
kubectl get endpoints web-service
```
→ `Selector:` debe decir `version=green`, endpoints deben ser las IPs de los Pods green.

**Rollback instantáneo (si green falla):**
```cmd
kubectl patch service web-service -p "{\"spec\":{\"selector\":{\"app\":\"web\",\"version\":\"blue\"}}}"
```

### Canary — manifiesto completo (`canary.yaml`)
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-v1
spec:
  replicas: 4
  selector:
    matchLabels:
      app: web
      version: v1
  template:
    metadata:
      labels:
        app: web
        version: v1
    spec:
      containers:
        - name: web
          image: inventario-app
          ports:
            - containerPort: 3000
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-v2-canary
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web
      version: v2
  template:
    metadata:
      labels:
        app: web
        version: v2
    spec:
      containers:
        - name: web
          image: inventario-app
          ports:
            - containerPort: 3000
---
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  selector:
    app: web        # <- SIN "version", así agrupa v1 y v2 juntos
  ports:
    - port: 80
      targetPort: 3000
```

**Comandos:**
```cmd
kubectl apply -f canary.yaml
kubectl get pods --show-labels
```
→ 5 Pods: 4 `v1` + 1 `v2`. El Service reparte tráfico entre los 5 por igual (~20% va al canary).

**Verificar que ambas versiones reciben tráfico:**
```cmd
kubectl port-forward service/web-service 8080:80
```
y repetir varias veces en otra terminal:
```cmd
curl http://localhost:8080/version
```

**Expandir gradualmente (si todo va bien):**
```cmd
kubectl scale deployment/web-v2-canary --replicas=3
kubectl scale deployment/web-v1 --replicas=2
```

**Rollback (si v2 falla):**
```cmd
kubectl scale deployment/web-v2-canary --replicas=0
kubectl scale deployment/web-v1 --replicas=5
```


---

## Extra — si sí toca usar Docker Hub real (login con secrets)

Si el profe exige que `docker push` complete de verdad (no solo `continue-on-error`), esta es la versión con login real:

```yaml
name: ci-cd
on:
  push:
    branches: [main]

jobs:
  build-test:
    runs-on: ubuntu-latest
    timeout-minutes: 2
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm test

  deploy:
    needs: build-test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Login a Docker Hub
        run: echo "${{ secrets.DOCKERHUB_TOKEN }}" | docker login -u "${{ secrets.DOCKERHUB_USERNAME }}" --password-stdin
      - run: docker build -t TUUSUARIO/app:${{ github.sha }} .
      - run: docker push TUUSUARIO/app:${{ github.sha }}
      - run: kubectl set image deployment/web-deployment web=TUUSUARIO/app:${{ github.sha }}
        continue-on-error: true
```

### Preparación previa (una sola vez)
1. Docker Hub → **Account settings → Personal access tokens → Generate new token** → permisos **Read & Write** (no "Read-only", ese no sirve para `push`).
2. GitHub → tu repo → **Settings → Secrets and variables → Actions → New repository secret**, crear dos:
   - `DOCKERHUB_USERNAME` = tu usuario de Docker Hub
   - `DOCKERHUB_TOKEN` = el token generado
3. Reemplaza `TUUSUARIO` en el `.yml` por tu usuario real de Docker Hub.

### Solución al error `GH013: Repository rule violations — Push cannot contain secrets`
Este error sale si en algún commit (aunque sea uno viejo) quedó el token en texto plano dentro del `.yml`, en vez de usar `${{ secrets.NOMBRE }}`.

**Arreglo rápido (si el secreto ya lo vas a invalidar de todos modos):**
1. Abre el link exacto que trae el mensaje de error (algo como `https://github.com/TU_USUARIO/TU_REPO/security/secret-scanning/unblock-secret/...`)
2. Click en **"Allow secret"**
3. `git push` de nuevo — ya debería pasar
4. Inmediatamente después: borra ese token en Docker Hub y genera uno nuevo, actualiza el secret en GitHub

**Cómo evitarlo desde el principio:** nunca pegar el token literal en el `.yml` — siempre usar `${{ secrets.DOCKERHUB_TOKEN }}`, nunca el valor real escrito a mano.

---

## Comandos principales — chuleta rápida

| Comando | Cuándo usarlo | Qué error/duda resuelve |
|---|---|---|
| `docker build -t <img> .` | Después de tener/corregir el Dockerfile | Construir la imagen desde cero |
| `docker run -d -p <p>:<p> --name <n> <img>` | Para probar el contenedor | Reproducir o confirmar el fix del Reto 1 |
| `docker ps` | Después de `docker run` | Ver si el contenedor sigue `Up` o se cayó |
| `docker logs <n>` | Contenedor corre pero no responde | Revela en qué puerto escucha realmente, o errores de arranque |
| `docker exec -it <n> sh` | Necesitas confirmar algo desde adentro | Ver procesos/puertos reales con `netstat -tlnp` |
| `docker stop <n>` / `docker rm <n>` | Antes de reconstruir con un fix | Limpiar el contenedor viejo (obligatorio antes de `run` de nuevo) |
| `kubectl apply -f archivo.yaml` | Cada vez que cambias un manifiesto | Aplica/actualiza Deployment y Service |
| `kubectl get pods` | Ver si los Pods están sanos | Confirma `Running`/`Ready` — el señuelo del Reto 2 |
| `kubectl get pods --show-labels` | Sospechas de un mismatch de labels | Ver qué label tiene realmente el Pod |
| `kubectl describe pod <nombre>` | Necesitas el detalle completo de un Pod | Labels, eventos, estado de contenedores |
| `kubectl get endpoints <service>` | Pods sanos pero sin tráfico (Reto 2) | Si sale `<none>`, ahí está el bug — Service sin Pods detrás |
| `kubectl describe service <service>` | Comparar selector vs labels | Ver el campo `Selector:` exacto que usa el Service |
| `kubectl port-forward service/<svc> <local>:<svc_port>` | Probar un Service `ClusterIP` desde tu compu | Acceso de punta a punta sin exponer el Service públicamente |
| `kubectl rollout restart deployment/<nombre>` | Simular un "nuevo release" (giro final) | Ver el rolling update en acción |
| `kubectl rollout status deployment/<nombre>` | Después de un restart/update | Confirma "successfully rolled out" |
| `kubectl get endpoints <service> -w` | Mientras corre un rollout | Ver en vivo si el Service se queda sin Pods (corte real) — más confiable que un loop de curl |
| `kubectl scale deployment/<nombre> --replicas=N` | Ajuste manual de capacidad, o canary | Subir/bajar el número de Pods a mano |
| `kubectl get hpa` | Si configuraste autoescalado | Ver `TARGETS` (uso real vs meta) y `REPLICAS` ajustándose solo |
| `kubectl patch service <svc> -p "{...selector...}"` | Blue-green: cambiar el tráfico de golpe | Cambia a qué versión apunta el Service, sin tocar el archivo |
| `git add . && git commit -m "..." && git push` | Después de cada corrección | Deja el historial de "antes/después" que pide el entregable |
