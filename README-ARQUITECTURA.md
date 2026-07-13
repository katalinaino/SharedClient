# Arquitectura de Despliegue — TechMarket Orders (demo-api)

## 1. Resumen

Este pipeline reemplaza el despliegue `RollingUpdate` original (sin validaciones) por una
estrategia **Blue-Green** sobre un clúster K3s (equivalente académico a Amazon EKS, según
aviso de la escuela), con **remediación automática en dos capas**:

1. Rollback a nivel de *pipeline* si el candidato nuevo no pasa el Health Gate.
2. Auto-sanación en tiempo real (*watchdog*) si el servicio ya en producción falla por
   cualquier motivo — incluido un error inyectado manualmente durante la defensa.

## 2. Por qué Blue-Green y no Canary

Se evaluaron ambas estrategias. Blue-Green se eligió porque:

- El corte de tráfico es **atómico**: un solo `kubectl patch` sobre el selector del Service
  público (`demo-api`), sin dependencias de un Ingress Controller con soporte de pesos
  (no disponible en esta instalación mínima de K3s).
- Permite **aislar completamente** al candidato del tráfico real durante la validación,
  usando un Service de *preview* (`demo-api-preview-<slot>`, ClusterIP interno) que el
  pipeline consulta desde el propio nodo K3s antes de exponerlo.
- El rollback es instantáneo: como el slot anterior nunca se apaga del todo (queda como
  *standby cálido* con 1 réplica), volver atrás es otro `kubectl patch`, no un nuevo
  despliegue.

## 3. Componentes

| Objeto | Rol |
|---|---|
| `Service/demo-api` | Única puerta de entrada pública. HAProxy académico solo conoce este nombre (`demo-api.default.svc.cluster.local:3000`). Su `selector.slot` es lo único que el pipeline cambia para mover tráfico. |
| `Deployment/demo-api-blue`, `Deployment/demo-api-green` | Los dos "colores". Siempre existen ambos; el inactivo queda como standby (1 réplica) para permitir rollback instantáneo. |
| `Service/demo-api-preview-blue`, `Service/demo-api-preview-green` | Services internos (ClusterIP) usados solo por el pipeline para el Health Gate. Nunca reciben tráfico de usuarios. |
| `Deployment/demo-api-watchdog` | Pod liviano con `kubectl` que monitorea el Service público cada 5s y ejecuta la reversión automática si detecta fallos sostenidos. |

## 4. Flujo del pipeline (`ci-cd.yaml`)

```
push a main / workflow_dispatch
        │
        ▼
 [Template] Build & Push  ──►  docker build + npm test + push a Docker Hub
        │
        ▼
 [Template] Deploy Blue-Green
   1. Detecta slot activo leyendo el selector del Service público
   2. Calcula slot candidato (el opuesto)
   3. Aplica la imagen nueva SOLO en el Deployment del slot candidato
   4. Espera rollout (kubectl rollout status)
   5. Health Gate: hasta 6 reintentos cada 5s contra el Service de
      preview del candidato (aislado, sin tráfico público)
        │
        ├── ✅ pasa ──► Promoción: patch del selector del Service
        │               público al slot candidato (100% del tráfico,
        │               corte atómico). Slot anterior queda en standby
        │               (1 réplica) por si el watchdog necesita revertir.
        │
        └── ❌ falla ──► Rollback automático: el tráfico NUNCA se movió.
                        Se escala el candidato defectuoso a 0 réplicas.
                        El job termina en rojo (falla intencional, sin
                        impacto a usuarios).
```

## 5. Remediación en tiempo real (Ítem 3 / "Prueba de Fuego")

El pipeline por sí solo **no** protege contra un error introducido *después* de un deploy
exitoso (por ejemplo, si el docente rompe algo manualmente en el clúster durante la
defensa). Para eso corre permanentemente `demo-api-watchdog`:

1. Cada 5 segundos consulta `/health` directamente contra la IP interna (ClusterIP) del
   Service público — el mismo endpoint que usan los usuarios reales.
2. Si detecta 3 fallos consecutivos:
   - Escala el slot standby a 2 réplicas.
   - Revierte el selector del Service público al slot anterior (instantáneo).
   - Escala a 0 el slot que falló.
3. Vuelve a monitorear con normalidad.

Esto significa que, sin importar qué tipo de error inyecte el docente (imagen rota,
proceso caído, `CrashLoopBackOff`, latencia/500 forzados, etc.), mientras se refleje en
`/health` dejando de responder `200`, el sistema se autorepara en como máximo
`3 × 5s = 15s` sin intervención humana.

Permisos: el watchdog corre bajo un `ServiceAccount` (`demo-api-watchdog`) con un `Role`
de alcance mínimo — solo puede leer/parchear el Service `demo-api` y escalar los
Deployments `demo-api-blue`/`demo-api-green` dentro del namespace (`k8s/watchdog/rbac.yaml`).

## 6. Variables de entorno dinámicas

El input `env-vars` del template de deploy acepta pares `CLAVE=VALOR` separados por coma
(ej. `LOG_LEVEL=debug,FEATURE_X=true`) y los inyecta como `env:` en el contenedor del slot
candidato, sin tocar el código fuente ni el Dockerfile. Se configuran al disparar el
pipeline manualmente (`workflow_dispatch`) o editando el `with.env-vars` de `ci-cd.yaml`.

## 7. Bootstrap (una sola vez, antes del primer pipeline)

```bash
scp -r k8s ubuntu@<IP_K3S>:/tmp/k8s-bootstrap
ssh ubuntu@<IP_K3S> '
  sudo k3s kubectl apply -f /tmp/k8s-bootstrap/deployment-blue.yaml
  sudo k3s kubectl apply -f /tmp/k8s-bootstrap/deployment-green.yaml
  sudo k3s kubectl apply -f /tmp/k8s-bootstrap/service.yaml
  sudo k3s kubectl apply -f /tmp/k8s-bootstrap/watchdog/rbac.yaml
  sudo k3s kubectl apply -f /tmp/k8s-bootstrap/watchdog/configmap-script.yaml
  sudo k3s kubectl apply -f /tmp/k8s-bootstrap/watchdog/deployment.yaml
'
```

## 8. Secrets y variables requeridas en GitHub (repo `SharedClient`)

| Tipo | Nombre | Contenido |
|---|---|---|
| Secret | `DOCKER_USERNAME` | Usuario de Docker Hub |
| Secret | `DOCKER_PASSWORD` | Token/contraseña de Docker Hub |
| Secret | `EA2_SSH_PRIVATE_KEY` | Llave privada SSH hacia el nodo K3s (usuario `ubuntu`) |
| Variable | `K3S_SERVER_PUBLIC_IP` | IP pública del servidor K3s |

## 9. Cómo simular la "Prueba de Fuego" para practicar

```bash
# Forzar que el slot activo empiece a fallar (ejemplo: imagen rota)
ssh ubuntu@<IP_K3S> 'sudo k3s kubectl set image deployment/demo-api-blue api=kata63/demo-api:no-existe -n default'

# Observar los logs del watchdog en vivo
ssh ubuntu@<IP_K3S> 'sudo k3s kubectl logs -f deployment/demo-api-watchdog -n default'
```

En ~15 segundos el watchdog debería revertir el Service hacia el otro slot automáticamente.

## 10. Evidencia de prueba real (ensayo de la "Prueba de Fuego")

Se ejecutó una simulación completa contra el clúster real (13-jul-2026) para validar
los tres ítems de punta a punta:

**Ítem 1 y 2 — Pipeline completo:**
Run manual en GitHub Actions (`CI/CD — TechMarket Orders`, run `#8`, commit `c7e5db4`):
Build → Deploy Blue-Green → Health Gate → Promoción, todo en verde (1m 16s).
Slot activo antes: `blue`. Slot promovido: confirmado vía
`kubectl get svc demo-api -o jsonpath='{.spec.selector.slot}'`.

**Ítem 3 — Auto-remediación en caliente (watchdog):**
1. Se rompió intencionalmente el slot activo (`blue`) apuntándolo a una imagen
   inexistente (`kata63/demo-api:version-que-no-existe`) y forzando el reemplazo
   de los pods sanos (simulando un error inyectado en producción).
2. El watchdog, monitoreando `/health` cada 5s, detectó 3 fallos consecutivos
   (`fails=1/3` → `2/3` → `3/3`).
3. Ejecutó la reversión automática: escaló `green` a 2 réplicas, parcheó el
   selector del Service `demo-api` de `blue` a `green`, y escaló `blue` a 0.
4. Tiempo total de detección + recuperación: **~15 segundos**, sin intervención
   humana.
5. Verificación post-remediación: `curl http://<IP>:30090/health` volvió a
   responder `{"ok":true,...}` y el Service quedó apuntando a `green`.

Log real del watchdog durante el incidente:
[watchdog] iniciado. namespace=default app=demo-api umbral=3 intervalo=5s
[watchdog] FALLO detectado en slot activo='blue' (código=000) -> fails=1/3
[watchdog] FALLO detectado en slot activo='blue' (código=000) -> fails=2/3
[watchdog] FALLO detectado en slot activo='blue' (código=000) -> fails=3/3
[watchdog] === AUTO-REMEDIACIÓN === umbral superado. Revirtiendo tráfico de 'blue' a 'green'.
deployment.apps/demo-api-green scaled
service/demo-api patched
deployment.apps/demo-api-blue scaled
[watchdog] tráfico revertido a 'green'. Slot defectuoso 'blue' escalado a 0
**Incidente durante las pruebas (documentado para transparencia):** el primer
intento de romper el Deployment vía `kubectl set image` no bastó, porque
Kubernetes conservó los pods sanos del ReplicaSet anterior (comportamiento
estándar de rollout, no un fallo del sistema). Fue necesario forzar el
reemplazo (`kubectl delete pod` + `kubectl scale rs --replicas=0` sobre el
ReplicaSet sano) para simular una caída real. Esto confirma que el propio
Kubernetes ya aporta una primera capa de resiliencia antes de que intervenga
el watchdog.
