# Paso 2: reestructurar los manifiestos con Kustomize (explicado simple)

## ¿Qué problema estábamos resolviendo?

Antes, `node-k8s` tenía 4 archivos planos en la raíz (`deployment.yaml`, `services.yaml`,
`namespace.yaml`, `ecr-cred-refresh.yaml`), todos con el namespace **fijo** escrito a mano:
`node-apic`. Eso funciona perfecto si solo existe un ambiente (producción), pero si
algún día queremos un ambiente extra (por ejemplo uno para revisar un Pull Request),
no podemos simplemente copiar esos archivos: tendrían el mismo nombre de namespace,
el mismo nombre de Deployment, el mismo nombre de Service... y chocarían entre sí.

Kustomize es la herramienta (viene incluida en `kubectl`, no instalamos nada nuevo)
que resuelve esto: nos deja tener **una sola copia "genérica"** de los manifiestos,
y luego generar variantes de esa copia para cada ambiente, cambiando solo lo que
cambia (el namespace, el tag de imagen, etc.) sin duplicar todo el YAML.

## Qué se creó, carpeta por carpeta

```
node-k8s/
├── deployment.yaml          <- archivos viejos, los originales. SIGUEN AHÍ, sin tocar.
├── services.yaml            <- producción sigue usando estos exactamente igual que antes.
├── namespace.yaml
├── ecr-cred-refresh.yaml
│
├── base/                     <- NUEVO: la copia "genérica", sin namespace escrito a mano
│   ├── kustomization.yaml    <- la "receta": lista qué archivos forman parte de la base
│   ├── deployment.yaml       <- igual al original, pero le quité la línea `namespace: node-apic`
│   ├── services.yaml         <- igual, sin namespace
│   └── ecr-cred-refresh.yaml <- igual, sin namespace
│
└── overlays/                 <- NUEVO: las "variantes" que sí tienen namespace
    └── production/
        ├── kustomization.yaml <- dice: "toma la base de ../../base, y ponle namespace: node-apic"
        └── namespace.yaml      <- el recurso que crea el namespace node-apic en sí
```

**Analogía:** `base/` es como una plantilla de Word sin rellenar (un contrato genérico).
Cada carpeta dentro de `overlays/` es una copia de ese contrato ya rellenada con los
datos específicos de un caso (en este caso, "producción" → namespace `node-apic`).
Más adelante, cuando agreguemos PRs, cada PR será una carpeta overlay nueva,
rellenando los mismos huecos con "namespace: node-api-pr-42", etc.

## Cómo verifiqué que no rompí nada

Corrí este comando, que le pide a `kubectl` que "renderice" (genere el YAML final)
a partir de la receta de `overlays/production`:

```bash
kubectl kustomize overlays/production
```

El resultado fue **byte por byte el mismo contenido** (mismos nombres, mismo
namespace `node-apic`, mismo Deployment, Service, CronJob, Role, etc.) que lo que
ya está corriendo en el cluster hoy. Es la prueba de que el cambio es solo de
organización de archivos, no de comportamiento.

## Por qué los archivos viejos siguen ahí (todavía)

ArgoCD (la Application `node-labs`) hoy está configurada para mirar la raíz del
repo (`path: .`), **no** la carpeta `overlays/production`. Como no recurse en
subcarpetas por defecto, ArgoCD sigue viendo y aplicando solo los 4 archivos
planos de siempre. La carpeta `base/`/`overlays/` existe en el repo pero ArgoCD
la ignora por completo — por eso producción no se vio afectada en ningún momento.

## Lo que falta (próximos pasos, todavía no hechos)

1. Crear un overlay nuevo, parametrizable, para Pull Requests (`overlays/pr-template`
   o similar), donde el namespace y el tag de imagen se generan a partir del número
   de PR.
2. Crear el `ApplicationSet` en ArgoCD con el generador de Pull Request, que use
   ese overlay para crear/borrar ambientes automáticamente.
3. Recién cuando todo esté probado: cambiar el `path` de la Application `node-labs`
   de `.` a `overlays/production`, y borrar los 4 archivos planos originales (ya
   no harán falta, `base/` + `overlays/production` los reemplaza).
4. Arreglar el `sed` del workflow de CI del repo `node`, porque hoy asume que
   `deployment.yaml` está en la raíz de `node-k8s`; cuando movamos todo de verdad,
   tendrá que apuntar a `base/deployment.yaml`.

Todo esto sigue viviendo en la rama `feature/preview-envs` (no en `main`), así que
nada de esto afecta producción hasta que decidamos hacer merge.
