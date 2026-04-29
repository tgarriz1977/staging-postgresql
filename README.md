# PostgreSQL compartido — colegio-staging

PostgreSQL 16 desplegado como StatefulSet en el namespace `postgresql`. Actúa como servidor de bases de datos compartido para todas las aplicaciones del cluster. Cada app tiene su propio usuario y base de datos aislada dentro de esta instancia.

## Archivos

| Archivo | Recurso | Descripción |
|---------|---------|-------------|
| `00-namespace.yaml` | Namespace | Namespace `postgresql` |
| `01-secret.yaml` | Secret `postgres-admin` | Credenciales del superusuario |
| `02-services.yaml` | Service x2 | `postgresql-headless` (StatefulSet) + `postgresql` (ClusterIP para apps) |
| `03-statefulset.yaml` | StatefulSet `postgresql` | Pod `postgresql-0`, PVC 50 Gi gp3 |

## Primer despliegue

### 1. Generar la password del superusuario

```bash
POSTGRES_PASSWORD=$(openssl rand -base64 24 | tr -d '/+=')
echo $POSTGRES_PASSWORD  # guardala en un lugar seguro
```

Editar `01-secret.yaml` y reemplazar `REEMPLAZAR_POSTGRES_PASSWORD`.

### 2. Aplicar los manifiestos

```bash
kubectl apply -f 00-namespace.yaml
kubectl apply -f 01-secret.yaml
kubectl apply -f 02-services.yaml
kubectl apply -f 03-statefulset.yaml
```

### 3. Verificar que levantó

```bash
kubectl get pods -n postgresql
kubectl exec -n postgresql postgresql-0 -- pg_isready -U postgres
```

## Conectarse al servidor (psql directo)

```bash
kubectl exec -it -n postgresql postgresql-0 -- psql -U postgres
```

## Agregar una nueva app

Cada vez que una nueva aplicación necesite una base de datos, ejecutar dentro del pod:

```bash
kubectl exec -n postgresql postgresql-0 -- psql -U postgres -c "
  CREATE DATABASE <app>;
  CREATE USER <app> WITH PASSWORD '<password>';
  GRANT ALL PRIVILEGES ON DATABASE <app> TO <app>;
"
```

Luego crear un Secret en el namespace de la app con esa password, y configurar:

```
DATABASE_HOST: postgresql.postgresql.svc.cluster.local
DATABASE_PORT: 5432
DATABASE_NAME: <app>
DATABASE_USERNAME: <app>
```

### Apps actuales

| App | Namespace | Base de datos | Usuario |
|-----|-----------|---------------|---------|
| Strapi | `strapi` | `strapi` | `strapi` |

## Operaciones comunes

### Ver todas las bases de datos

```bash
kubectl exec -n postgresql postgresql-0 -- psql -U postgres -c "\l"
```

### Ver todos los usuarios

```bash
kubectl exec -n postgresql postgresql-0 -- psql -U postgres -c "\du"
```

### Backup manual de una base de datos

```bash
kubectl exec -n postgresql postgresql-0 -- \
  pg_dump -U postgres <app> > backup-<app>-$(date +%Y%m%d).sql
```

### Restaurar un backup

```bash
kubectl exec -i -n postgresql postgresql-0 -- \
  psql -U postgres <app> < backup-<app>-YYYYMMDD.sql
```

## Recursos y límites

| | Valor |
|-|-------|
| Image | `postgres:16-alpine` |
| CPU request | 200m |
| Memory request | 512 Mi |
| Memory limit | 2 Gi |
| Storage | 50 Gi gp3 (ReadWriteOnce) |
| PVC | `data-postgresql-0` (creado por `volumeClaimTemplates`) |

## Notas

- **Sin HA**: réplica única. Para alta disponibilidad considerar CloudNativePG o RDS.
- **Backups**: Velero hace snapshot del PVC. Para backups lógicos consistentes usar `pg_dump` antes de la ventana de Velero.
- **TLS entre apps**: deshabilitado dentro del cluster (tráfico privado). Habilitarlo si el threat model lo requiere.
