# go-frontend-test

Frontend en Go desplegado por Argo CD.

- Imagen: `docker.io/guslopezc/go-frontend-test`
- Namespace / AppProject / Application: `go-frontend-test`
- Ingress: `https://go-frontend-test.local`

## Local

```bash
go run .
```

Servidor stdlib que sirve `public/` (SPA) y expone `/health` en el puerto `PORT` (defecto 8080).

## CI

`.github/workflows/ci.yml` construye y publica la imagen y actualiza la tag en
`deployment.yaml`; Argo CD (sync automatico con selfHeal) desplega el cambio.