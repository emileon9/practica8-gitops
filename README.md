# sa-platform — repositorio GitOps (plantilla)

Este directorio es la plantilla exacta que debes copiar como el
**contenido inicial** del repositorio GitOps independiente que la
Práctica 8 exige (ver
[../docs/GITOPS.md, sección 5](../docs/GITOPS.md#5-qué-debe-crear-manualmente-el-estudiante-en-github)
para los pasos manuales de creación en GitHub).

## Por qué este repo existe y qué NO contiene

Este repositorio es deliberadamente pequeño: describe **qué está
desplegado**, no cómo se construye. Eso significa dos cosas: el tag de
imagen de cada componente por ambiente, y los secretos sellados. Todo lo
demás (charts de Helm, templates, recursos, probes, políticas) vive en el
repositorio de código (`P8/helm/*`), y las Applications de ArgoCD combinan
ambas fuentes (ver `P8/argocd/applications/*.yaml`, campo `sources`).

```
apps/                        tag de imagen por servicio y ambiente
├── gateway/
│   ├── values-dev.yaml     { image: { tag: "" } }
│   └── values-prod.yaml
├── ms-users/
├── ms-products/
├── ms-orders/
├── ms-notifications/
├── cronjob-heartbeat/
└── cronjob-summary/

secrets/                     credenciales selladas (ciphertext)
├── sealedsecret-db-credentials.yaml
└── sealedsecret-broker-credentials.yaml
```

### Por qué los secretos viven aquí y no en el chart

Un `SealedSecret` es ciphertext asimétrico: solo el controller que corre en
el clúster de destino puede descifrarlo, con una llave privada que nunca
sale de ahí. Por eso es seguro versionarlo en un repositorio público — y
por eso su lugar correcto es este repositorio, que representa el estado
desplegado, y no el chart de Helm, que describe la forma de la aplicación
independientemente del entorno.

Nunca debe aparecer aquí una contraseña en claro. Las que están selladas se
generaron aleatoriamente y se aplicaron al servicio correspondiente en el
mismo paso en que se sellaron.

## Cómo se actualiza

**Nunca a mano en el flujo normal.** `.github/workflows/gitops-update.yml`
(en el repositorio de código) abre un Pull Request contra este repositorio
cada vez que se publica un tag de Git `vX.Y.Z`: build → Trivy → SBOM →
push a GHCR → cosign sign → PR aquí actualizando `apps/<servicio>/values-
prod.yaml`. Al hacer merge del PR, ArgoCD detecta el cambio y sincroniza.

## Pasos para activarlo (una sola vez, manual)

1. Crear este repositorio en GitHub (vacío) y copiar aquí el contenido de
   `P8/gitops-repo-template/` (esta carpeta) como commit inicial.
2. En el repositorio de código, crear el secret `GITOPS_REPO_TOKEN` (token
   con permiso de escritura solo sobre este repo) y las variables de
   repositorio `GITOPS_REPO_OWNER` / `GITOPS_REPO_NAME`
   (`Settings → Secrets and variables → Actions`).
3. Actualizar `P8/argocd/applications/*.yaml` y
   `P8/argocd/project/sa-platform-project.yaml` en el repo de código,
   reemplazando `<GITOPS_REPO_OWNER>/<GITOPS_REPO_NAME>` por la URL real
   de este repositorio.
4. Instalar ArgoCD en el clúster de la demo y aplicar
   `P8/argocd/project/sa-platform-project.yaml` y las 8 Applications de
   `P8/argocd/applications/`.
