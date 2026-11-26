# ars-humano-gitops

Estado deseado del despliegue de **Health Authorizations API** en Kubernetes.
El código de la aplicación vive en [ars-humano-api](https://github.com/zaizo01/ars-humano-api); este
repositorio solo declara **qué versión corre en cada entorno y con qué configuración**.

## Cómo funciona

```
 ars-humano-api                    ars-humano-gitops                     Clúster
┌─────────────────────┐  commit   ┌──────────────────────────┐  sync   ┌────────────────┐
│ CI: build, pruebas, │ ────────► │ overlays/dev  (imagen)   │ ──────► │ Argo CD aplica │
│ imagen → ghcr.io    │    PR     │ overlays/prod (imagen)   │         │ lo que dice Git│
└─────────────────────┘ ────────► └──────────────────────────┘         └────────────────┘
```

- **Push a `develop`** en la API → el pipeline publica la imagen `sha-<commit>` y hace commit aquí
  cambiando la versión de **dev**. Argo CD sincroniza automáticamente.
- **Tag `vX.Y.Z`** en la API → el pipeline abre un **pull request** aquí cambiando la versión de **prod**.
  Al aprobarlo y fusionarlo, alguien con permisos sincroniza en Argo CD.
- **Rollback:** revertir el commit en este repositorio. Git es la única fuente de verdad y el historial
  es la auditoría de qué se desplegó, cuándo y quién lo aprobó.
- El pipeline de la API **nunca** tiene credenciales del clúster: solo escribe en este repo.

## Estructura

```
apps/health-authorizations-api/
  base/                      Manifiestos comunes
    deployment.yaml          Rolling update sin pérdida de capacidad, probes, contenedor endurecido
    migration-job.yaml       Migraciones como hook PreSync (antes de actualizar las réplicas)
    service.yaml, ingress.yaml, hpa.yaml, pdb.yaml, networkpolicy.yaml, configmap.yaml
  overlays/
    dev/                     1 réplica, PostgreSQL y RabbitMQ dentro del clúster, usuarios demo
    prod/                    3–10 réplicas repartidas por zona/nodo, TLS, secretos vía External Secrets
argocd/
  project.yaml               Restringe repositorio y namespaces permitidos
  applications/              dev (sync automático + selfHeal) y prod (sync manual)
```

## Decisiones

| Tema | Decisión |
|---|---|
| Herramienta de plantillas | **Kustomize** (incluido en `kubectl`): parches sobre YAML plano, sin lenguaje de plantillas. |
| Migraciones | Job con la misma imagen (`--migrate`) como hook `PreSync`. Si falla, el despliegue se detiene y la versión anterior sigue atendiendo. Las réplicas nunca migran al arrancar. |
| Disponibilidad | `maxUnavailable: 0`, `PodDisruptionBudget`, `topologySpreadConstraints` y `preStop` para no cortar solicitudes en curso. |
| Escalado | HPA por CPU (70 %), con ventana de estabilización para evitar oscilaciones. |
| Probes | `startup` y `live` → `/health/live`; `ready` → `/health/ready` (incluye PostgreSQL). RabbitMQ no afecta la readiness: los eventos esperan en el outbox. |
| Seguridad | Pod Security `restricted`, usuario no root, sistema de archivos de solo lectura, sin capabilities, sin token de service account, NetworkPolicy que solo admite tráfico del ingress. |
| Secretos | dev: generados por Kustomize con valores de ejemplo **solo para entornos locales**. prod: `ExternalSecret` → Azure Key Vault; en Git no hay valores. |
| Dependencias | dev: PostgreSQL y RabbitMQ en el clúster para poder levantarlo completo localmente. prod: servicios administrados. |

## Probar en un clúster local

Requisitos: Kubernetes local (Docker Desktop, kind o minikube) con un ingress controller nginx.

```bash
# Sin Argo CD: aplicar el overlay de dev directamente
kubectl apply -k apps/health-authorizations-api/overlays/dev
kubectl -n ars-humano-dev get pods -w

# Con Argo CD instalado: registrar proyecto y aplicaciones
kubectl apply -k argocd/
```

> La imagen se publica en `ghcr.io/zaizo01/ars-humano-api`. Si el paquete es privado, crear un
> `imagePullSecret` en el namespace o hacer público el paquete en GitHub.

Sin Argo CD, el Job de migración corre junto con el Deployment (el hook solo ordena la ejecución en Argo CD);
la API reintenta la conexión mientras tanto.

## Validación

Cada push y pull request ejecuta `kustomize build` + `kubeconform` sobre cada overlay
(`.github/workflows/validate.yml`), así un cambio inválido no llega al clúster.
