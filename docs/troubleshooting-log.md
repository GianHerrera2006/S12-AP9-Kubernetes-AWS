# Bitácora de Troubleshooting - S12 AP9 Kubernetes en AWS

## Incidente 1: Backend en estado NotReady / CrashLoopBackOff y falla en Readiness/Liveness Probes

### 1. Síntomas observados
- El pod `s12-database` levantó en estado `1/1 Running` en el nodo Worker (`ip-172-31-16-91`), escuchando en el puerto `1433`.
- El pod `s12-backend` permanecía en `0/1 Running` y posteriormente entró en `CrashLoopBackOff`.
- `endpoints/backend-service` aparecía vacío, impidiendo conexiones externas e internas al puerto NodePort `30080`.
- Los eventos (`kubectl get events -n practica-s12`) reportaron:
  - `Readiness probe failed: Get "http://10.42.1.3:8080/ready": context deadline exceeded`
  - `Liveness probe failed: Get "http://10.42.1.3:8080/health": context deadline exceeded`
  - `Killing: Container backend failed liveness probe, will be restarted`

### 2. Diagnóstico y Causa Raíz
1. **Inconsistencia de puerto en el Service (`db-service`):**
   - El ConfigMap `app-config` configuraba `DB_HOST=db-service` y `DB_PORT=1433`, mientras que `04-app-service.yaml` exponía `db-service` en el puerto `5432` (PostgreSQL) en lugar de `1433` (SQL Server).
2. **Bloqueo de red entre nodos en AWS Security Group:**
   - La red overlay de K3s (Flannel VXLAN, puerto UDP `8472`) estaba bloqueada por las reglas de entrada del Security Group de EC2, impidiendo la resolución DNS hacia CoreDNS en el nodo Master (`10.42.0.3`) desde los pods ubicados en el nodo Worker.
3. **Bloqueo del servidor HTTP single-thread:**
   - Al quedar bloqueada la petición `/ready` intentando abrir el socket hacia `db-service:1433` (timeout de 3s), el hilo único de `HTTPServer` en Python no respondía a tiempo a la sonda `/health` (timeout de 1s), provocando el reinicio continuo del contenedor.

### 3. Resolución aplicada
1. Se actualizó el manifiesto `k8s/04-app-service.yaml` cambiando el puerto de `db-service` de `5432` (`postgres`) a `1433` (`sqlserver`) y se aplicó con `kubectl apply -f ~/k8s/04-app-service.yaml`.
2. Se habilitaron las reglas de tráfico interno en el Security Group de AWS EC2 para permitir la comunicación entre los nodos Master y Worker.
3. Se ejecutó `kubectl rollout restart deployment/s12-backend -n practica-s12` para reiniciar el despliegue sin esperar el temporizador exponencial de `CrashLoopBackOff`.

### 4. Verificación
- `kubectl get pods,endpoints -n practica-s12` confirmó ambos pods en `1/1 Running` y `endpoints/backend-service` activo en `10.42.1.3:8080`.
- Pruebas funcionales HTTP con `curl`:
  - `GET http://localhost:30080/` $\rightarrow$ `200 OK` (`S12 AP9 Cloud Application`)
  - `GET http://localhost:30080/health` $\rightarrow$ `200 OK` (`Backend is alive`)
  - `GET http://localhost:30080/ready` $\rightarrow$ `200 OK` (`SQL Server connection available`)