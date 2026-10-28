# POC · Activación de Servicios con Kafka

Trabajo Práctico N° 6 — **Sistemas Distribuidos 2026**
Ingeniería en Sistemas de Información

Prueba de concepto de una **arquitectura orientada a eventos (EDA)** donde un cliente
contrata un plan y un único evento dispara, en paralelo, la facturación, el
aprovisionamiento, la notificación y el registro en CRM. Ningún servicio llama
directamente a otro: todo se coordina por Kafka.

---

## Requisitos

- Docker Desktop con Docker Compose v2 (probado con Docker 29 / Compose v5).
- 4 GB de RAM disponibles para Docker (Kafka + Mongo + 5 servicios NestJS + UI).
- Puertos libres: `3000`, `5173`, `8080`, `8025`, `9092`, `27017`.

## Levantar todo (un solo comando)

```bash
docker compose up --build -d
```

La primera vez tarda unos minutos porque construye las imágenes.
Cuando termine:

| Herramienta | URL | Para qué |
|---|---|---|
| **Demo UI** | http://localhost:5173 | Contratar planes y ver la timeline en vivo |
| **Kafka UI** | http://localhost:8080 | Topics, mensajes, particiones, consumer groups y lag |
| **Mailhog** | http://localhost:8025 | Bandeja de entrada de los emails simulados |
| **API REST** | http://localhost:3000 | `POST /activations`, `GET /activations/:id` |
| MongoDB | `localhost:27017` | Inspección con Compass (`directConnection=true`) |

Para ver los logs:

```bash
docker compose logs -f
docker compose logs -f activation-api billing-service
```

Para apagar:

```bash
docker compose down        # conserva los datos
docker compose down -v     # borra también los volúmenes (demo limpia)
```

> Si levantaron el perfil `loyalty`, incluyan el perfil al apagar:
> `docker compose --profile loyalty down -v`.

## Estructura del repositorio

```text
.
├── apps/
│   ├── activation-api/          REST + WebSocket + agregador de la Saga
│   ├── billing-service/         Facturación y compensación
│   ├── provisioning-service/    Aprovisionamiento simulado (1-3 s)
│   ├── notification-service/    Emails a Mailhog
│   ├── crm-analytics-service/   Consumidor pasivo de todos los topics
│   ├── loyalty-service/         Consumidor nuevo con replay (perfil `loyalty`)
│   └── demo-ui/                 React + Vite + socket.io-client
├── libs/
│   ├── contracts/               Topics, nombres de eventos, sobre común y tipos
│   └── kafka-toolkit/           Envelope, publisher, idempotencia, reintentos y DLQ
├── docker/
│   ├── create-topics.sh         Crea 4 topics + 4 DLQ (3 particiones, 7 días)
│   └── mongo-init.sh            Inicializa MongoDB como replica set de 1 nodo
├── docs/
│   ├── architecture.md          Decisiones, flujos, modelo de datos y RNF
│   ├── demo-script.md           Guion de los 5 escenarios (15 minutos)
│   └── defensa-oral.md          Conceptos y preguntas típicas del oral
└── docker-compose.yml
```

## Los 5 escenarios de la demo

El guion completo con comandos y resultados esperados está en
[`docs/demo-script.md`](docs/demo-script.md).

1. **Camino feliz:** contratar sin fallo → 4 eventos verdes + email en Mailhog.
2. **Fallo y compensación:** fallo en provisioning → `ProvisioningFailed`,
   `ActivationFailed` y `BillingAccountCancelled`.
3. **Servicio caído:** detener notification, crear activaciones y ver el *lag*;
   al volver, procesa todo sin perder mensajes.
4. **Escalar:** `docker compose up -d --scale billing-service=2` → Kafka reparte
   las particiones entre las dos instancias.
5. **Replay del log:** levantar `loyalty-service`, que lee `activation.events`
   desde el inicio con un consumer group nuevo.

## Documentación para estudiar

- [`docs/architecture.md`](docs/architecture.md): qué hace cada archivo y por qué.
- [`docs/defensa-oral.md`](docs/defensa-oral.md): explicaciones cortas y preguntas
  probables del examen oral, con las respuestas.

## Notas rápidas

- `kafka-init` y `mongo-init` terminan con estado **Exited (0)**. Es normal: son
  contenedores de inicialización, no servidores.
- Los grupos de consumo se llaman exactamente `activation-api-group`,
  `billing-service-group`, etc. (configuramos `postfixId: ''` para que NestJS no
  agregue el sufijo `-server`).
- La UI usa `http://localhost:3000` por defecto. Se puede cambiar con
  `VITE_API_URL` en el build de `apps/demo-ui`.
