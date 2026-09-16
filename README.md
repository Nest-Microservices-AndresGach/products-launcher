# Tienda — Arquitectura de Microservicios con NestJS

Orquestador de una tienda online construida como sistema distribuido: un API Gateway
y cuatro microservicios que se comunican de forma asíncrona sobre **NATS**, cada uno
con su propia base de datos, empaquetados en Docker y con manifiestos de Kubernetes
para desplegarlos en un clúster.

> **Desarrollado entre marzo y septiembre de 2025.** El grueso de la arquitectura
> (gateway, NATS, submódulos, Docker y Kubernetes) se construyó entre agosto y septiembre
> de 2025. Los commits de 2026 corresponden a mantenimiento: actualización de
> dependencias por vulnerabilidades y reorganización de los submódulos.

---

## Arquitectura

```
                            ┌──────────────┐
          HTTP / REST       │              │
   Cliente ───────────────► │ client-gateway│  :3000
                            │              │
                            └──────┬───────┘
                                   │
                        ┌──────────▼──────────┐
                        │        NATS         │  :4222
                        │   (message broker)  │
                        └──┬────┬────┬────┬───┘
                           │    │    │    │
         ┌─────────────────┘    │    │    └─────────────────┐
         │             ┌────────┘    └────────┐             │
         ▼             ▼                      ▼             ▼
   ┌──────────┐  ┌──────────┐          ┌──────────┐  ┌────────────┐
   │ auth-ms  │  │products-ms│          │ orders-ms│  │ payments-ms│
   │ MongoDB  │  │  SQLite   │          │PostgreSQL│  │   Stripe   │
   └──────────┘  └──────────┘          └──────────┘  └─────┬──────┘
                                                            │
                                                   webhook  │
                                                   (HTTP)   ▼
                                                        Stripe API
```

**Flujo de una compra:** el cliente pide una orden al gateway → `orders-ms` valida los
productos contra `products-ms` → solicita una sesión de pago a `payments-ms` → Stripe
confirma vía webhook → `payments-ms` notifica a `orders-ms` y la orden pasa a pagada.

Solo el gateway y el webhook de pagos están expuestos al exterior. El resto de los
servicios viven detrás de NATS y no tienen puerto público.

## Servicios

| Servicio | Responsabilidad | Persistencia | Repositorio |
|---|---|---|---|
| **client-gateway** | Único punto de entrada HTTP. Enruta a los microservicios, valida DTOs y protege rutas con JWT | — | [client-gateway](https://github.com/Nest-Microservices-AndresGach/client-gateway) |
| **auth-ms** | Registro, login y verificación de tokens. Hash de contraseñas con bcrypt | MongoDB | [auth-ms](https://github.com/Nest-Microservices-AndresGach/auth-ms) |
| **products-ms** | CRUD de productos con paginación y borrado lógico | SQLite | [products-microservice](https://github.com/Nest-Microservices-AndresGach/products-microservice) |
| **orders-ms** | Creación y consulta de órdenes, validación de productos y estados de pago | PostgreSQL | [orders-microservice](https://github.com/Nest-Microservices-AndresGach/orders-microservice) |
| **payments-ms** | Sesiones de pago de Stripe y recepción del webhook de confirmación | — | [payments-microservice](https://github.com/Nest-Microservices-AndresGach/payments-microservice) |

## Stack

NestJS · TypeScript · NATS · Prisma · PostgreSQL · MongoDB · SQLite · Stripe ·
Docker · Docker Compose · Kubernetes · Google Cloud Build

---

## Puesta en marcha

Los microservicios están enlazados como **git submodules**, así que el clon debe ser
recursivo:

```bash
git clone --recurse-submodules https://github.com/Nest-Microservices-AndresGach/products-launcher.git
cd products-launcher
```

Si ya lo clonaste sin submódulos:

```bash
git submodule update --init --recursive
```

Después:

```bash
cp .env.template .env      # y completa los valores
docker compose up --build
```

El gateway queda en `http://localhost:3000`.

### Producción

```bash
docker compose -f docker-compose.prod.yml build
```

### Kubernetes

En `k8s/tienda` están los manifiestos para desplegar el sistema en un clúster:
Deployments para los cinco servicios y NATS, Services para los que necesitan ser
alcanzables, e Ingress para el gateway y para el webhook de Stripe. Las imágenes se
publican en Artifact Registry de Google Cloud.

```bash
kubectl apply -f k8s/tienda/templates -R
```

La carpeta está estructurada como un chart de Helm, pero los manifiestos son YAML
plano sin parametrizar: `values.yaml` está vacío y las plantillas no usan variables.
Parametrizarlos es una mejora pendiente.

Ver [K8s.README.md](./K8s.README.md) para el detalle del despliegue.

---

## Trabajar con los submódulos

Al tocar un microservicio, **primero commit y push en el submódulo, después en este
repositorio**. Si se hace al revés, el puntero del launcher queda apuntando a un
commit que no existe en el remoto y hay que resolverlo a mano.

```bash
# dentro del submódulo
git add . && git commit -m "..." && git push

# de vuelta en el launcher
git add <submodulo> && git commit -m "chore: update submodule reference" && git push
```

Para traer las últimas referencias de todos:

```bash
git submodule update --remote
```

---

## Sobre el proyecto

Construido siguiendo el curso de microservicios con NestJS de DevTalles. El objetivo
fue entender de primera mano los problemas reales de un sistema distribuido:
comunicación asíncrona entre servicios, consistencia cuando cada uno tiene su propia
base de datos, integración con un tercero que responde por webhook, y el empaquetado
necesario para llevarlo a un orquestador.

**Limitaciones conocidas**, por transparencia: no hay tests automatizados, los
manifiestos de Kubernetes no están parametrizados, y `products-ms` usa SQLite dentro
del contenedor, por lo que sus datos no persisten entre reinicios del pod. El foco
estuvo en la arquitectura de mensajería, no en dejarlo listo para producción.
