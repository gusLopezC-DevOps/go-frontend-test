# go-frontend-test

Frontend en Go (`net/http` + distroless) **generado desde el template
`go-frontend` de Backstage** y desplegado en el cluster k3s mediante
Backstage → control-plane → Argo CD.

- Imagen: `docker.io/guslopezc/go-frontend-test` (tag = `${{ github.sha }}`)
- Namespace / AppProject / Application: `go-frontend-test`
- Ingress (Kong): `https://go-frontend-test.local`
- Catálogo Backstage: `component:default/go-frontend-test` (owner `group:default/development`)

## Cómo se creó (proceso real)

1. Template `go-frontend` en Backstage (`backstage-gitops/assets/templates/go-frontend`).
2. `fetch:template` → `publish:github` (repo **público**, rama `main`, commit inicial).
3. CI del repo: `go vet` + `go build` + `docker build/push` + autocommit `ci: bump imagen`.
4. `github:actions:dispatch` → `control-plane/workflow-alta-pr.yml` → `alta-workload.py`
   → PR «Alta workload go-frontend-test» (**PR #4**) → merge.
5. `root-app` de Argo (prune + selfHeal) crea `AppProject` + `Application go-frontend-test`
   (Synced + Healthy) y el namespace con sus manifests.
6. `catalog:register` con URL raw → entidad visible en el catálogo.

## Estructura

| Fichero | Descripción |
|---|---|
| `main.go` | servidor stdlib: sirve `public/` y expone `/health` (puerto `PORT`, default 8080) |
| `go.mod` | solo stdlib (`net/http`) |
| `Dockerfile` | multi-stage → distroless |
| `deployment.yaml` | Deployment/port 8080, imagen gestionada por CI |
| `service.yaml` / `ingress.yaml` | Service + Ingress `ingressClassName: kong`, host `go-frontend-test.local` |
| `catalog-info.yaml` | entidad de catálogo (`type: website`, `owner: development`) |
| `.github/workflows/ci.yml` | build + push imagen + bump imagen (dispara sync de Argo) |
| `public/index.html`, `mkdocs.yml`, `README.md` | contenido + TechDocs |

## CI (`ci.yml`)

```
push a main → go vet + go build → docker buildx+push (tag <sha>)
→ sed image en deployment.yaml → autocommit 'ci: bump imagen' (si cambió) → Argo selfHeal reaplica
```

> Nota: el autocommit usa contenido de `deployment.yaml` con **comillas dobles**
> para el sed (no rompe el YAML).

## Evolución de un cambio

```text
commit/push → CI (build+push+tag sha) → bump imagen en deployment.yaml
→ Argo (app go-frontend-test, selfHeal) → nuevo pod → readiness → Healthy
```

La Application de Argo lee el repo con `path: .`, `recurse: true` y
`exclude: catalog-info.yaml` (el componente del catálogo no entra en el diff).

## Local

```bash
go run .
```

## Verificación

```bash
curl https://go-frontend-test.local/health     # {"status":"ok"}
curl http://go-frontend-test.local/            # index.html
```

`/etc/hosts` de los clientes: `192.168.100.77 go-frontend-test.local`.

## Baja

1. Borrar el repo y su `catalog-info.yaml` del catálogo (location + entidad).
2. `control-plane`: eliminar `apps/go-frontend-test.yaml` + `projects/go-frontend-test.yaml`
   + kustomization → push → force-sync `root-app` → se limpia el namespace.

Referencias del proceso completo: `backstage-gitops/docs/proceso.md`.