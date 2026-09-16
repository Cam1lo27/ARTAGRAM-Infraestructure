# Artagram — red social colaborativa para artistas visuales

Plataforma para publicar obras con su proceso, formar comunidades con un lienzo
compartido en tiempo real, y jugar "modo fiesta" (rondas cronometradas donde todos
pintan con el mismo pincel). Implementa las 21 historias *Must have* del Release 1
(MVP) del story map del curso — ver [`docs/gherkin-coverage.md`](docs/gherkin-coverage.md)
para la trazabilidad completa HU → escenario → prueba automatizada.

## Tecnologías

| Capa | Stack |
|---|---|
| **Backend** | Java 21, Spring Boot 3.3.4 (Web, WebSocket/STOMP, Data JPA, Security, Validation, Actuator, Mail), PostgreSQL 16 + Flyway, Redis 7 + Redisson (locks, rate-limiter y Pub/Sub para tiempo real multi-instancia), JWT, springdoc-openapi (Swagger), Azure Blob Storage SDK (almacenamiento intercambiable), Micrometer + Prometheus, Logback JSON. Detalle completo en el [README de `ARTAGRAM-BackEnd`](https://github.com/Cam1lo27/ARTAGRAM-BackEnd). |
| **Frontend** | React 19 + TypeScript, Vite 8, React Router 7, Zustand, Axios, `@stomp/stompjs`, Tailwind CSS 3, Framer Motion. Detalle completo en el [README de `ARTAGRAM-FrontEnd`](https://github.com/Cam1lo27/ARTAGRAM-FrontEnd). |
| **Infraestructura** | Docker + Docker Compose, GitHub Container Registry (ghcr.io) para las imágenes, GitHub Actions (CI/CD), Prometheus + Grafana (observabilidad) |
| **Testing** | JUnit 5, Mockito, Testcontainers (Postgres + Redis reales), cliente STOMP real, JaCoCo, SpotBugs |

## Arquitectura

**Backend: Java 21 + Spring Boot 3** (Web, WebSocket/STOMP, Data JPA, Security, Redis,
Actuator). Spring tiene soporte de primera clase para STOMP sobre WebSocket con
interceptores de canal, lo que permite autorizar cada `SUBSCRIBE`/mensaje por comunidad
o sala (no solo al conectar) — encaja directo con HU-11 y HU-22.

- **PostgreSQL** (Flyway) es el estado autoritativo: usuarios, comunidades, trazos
  append-only, salas de fiesta, cuotas, eventos de negocio.
- **Redis + Redisson** para coordinación distribuida: un `DistributedLockService`
  propio (compare-and-set con `SET NX PX` + Lua para renovar/liberar) implementa el
  bloqueo de edición de trazo (HU-15, TTL 30s) y la exclusión mutua del borrador del
  modo fiesta (HU-20, TTL 10s) — se eligió sobre `Redisson RLock` porque `RLock` está
  pensado para reentrancia por hilo de la JVM, no para un flujo adquirir/renovar/liberar
  entre peticiones HTTP independientes. También hay un `RRateLimiter` (100 ops/s por
  usuario en el banner, HU-16) y el reloj de ronda del modo fiesta vive en Redis para
  que sea consistente entre instancias.
- **Contadores atómicos**: los likes se incrementan/decrementan con `UPDATE` JPQL en
  bloque (`@Modifying(clearAutomatically = true)`), nunca leer-sumar-escribir — por eso
  el contador queda exacto con 200 usuarios dando like a la vez (ver `LikeConcurrencyIT`).
- **Secuencia de trazos**: cada lienzo/banner-región/sala tiene una fila contadora con
  `SELECT ... FOR UPDATE`, así el orden de los trazos es consistente sin depender de un
  segundo sistema.
- **JWT** compartido entre REST y el *handshake* de STOMP: el token viaja como
  `?token=` en la URL del WebSocket (un navegador no puede mandar headers propios en el
  handshake nativo) y se valida en `JwtHandshakeInterceptor` + `JwtHandshakeHandler`
  *antes* de aceptar la conexión — el `Principal` resultante persiste durante toda la
  sesión STOMP (a diferencia de intentar setearlo en un `ChannelInterceptor` sobre el
  frame `CONNECT`, que no persiste a los frames siguientes).
- **Broker STOMP multi-instancia**: `SimpleBroker` en memoria por instancia, pero ningún
  componente de dominio lo usa directo — todos pasan por `RealtimeBroadcaster`, que
  publica cada mensaje en un `RTopic` de Redis y cada instancia (incluida la que lo
  originó) lo recibe y hace la entrega local real. Se eligió Redis Pub/Sub en vez de un
  *broker relay* dedicado (RabbitMQ/ActiveMQ, la forma "de libro" vía
  `enableStompBrokerRelay`) porque reusa el Redis que el proyecto ya tiene para locks y
  rate-limiting, sin infraestructura ni costo nuevo — correr el backend con más de una
  instancia detrás de un load balancer ya no rompe el tiempo real.
- **Almacenamiento intercambiable**: `StorageService` (validación + firma HMAC) delega en
  un `StorageProvider` — disco local por defecto, o Azure Blob Storage activándolo con
  `STORAGE_PROVIDER=blob` (sin tocar código), para que los archivos subidos sobrevivan un
  redeploy/reinicio en producción.
- **Observabilidad**: Micrometer + `/actuator/prometheus`; logs JSON (Logback +
  `logstash-logback-encoder`) con `traceId` propagado en HTTP y STOMP vía MDC; una tabla
  de eventos de negocio (`EventoNegocio`) alimenta los 4 KPIs (HU-37) calculados bajo
  demanda con fórmula, periodo y tendencia.

**Frontend: React 19 + TypeScript + Vite.** Canvas 2D nativo (no WebGL) para los tres
lienzos (estudio colaborativo, banner mural, modo fiesta) — con `globalCompositeOperation`
y `filter: blur()` alcanza para simular textura de pincel (lápiz/marcador/acuarela) sin
la complejidad de WebGL, y es más que suficiente en rendimiento para el trazo por puntos
que maneja la app. `@stomp/stompjs` para tiempo real (una sola conexión STOMP por sesión,
con reconexión y re-suscripción automática). Zustand para la sesión, Tailwind CSS +
Framer Motion para la estética de "estudio creativo" oscuro.

**Patrón cliente-servidor para tiempo real**: el cliente se suscribe al topic de STOMP
*antes* de pedir el snapshot REST (bufferea lo que llegue mientras tanto), y recién
después reproduce el buffer filtrando lo que ya vino en el snapshot — evita perder
trazos en la ventana de carrera entre "me suscribo" y "pido el estado actual". El reloj
del modo fiesta es siempre autoritativo del servidor (`TICK` cada 500ms); el cliente solo
interpola visualmente entre ticks para que la cuenta regresiva se vea fluida.

### Estructura

```
ARTAGRAM-BackEnd/          Spring Boot (Maven) — un paquete por módulo de dominio
  src/main/java/.../{auth,profile,community,canvas,party,chat,feed,billing,observability,realtime,common}
  src/test/java/...        unitarias (JUnit5+Mockito) + integración (Testcontainers + STOMP real)
ARTAGRAM-FrontEnd/         React + TS + Vite
  src/features/{auth,profile,community,canvas,party,chat,feed,admin}
  src/lib/                 cliente STOMP, cliente HTTP, motor de canvas, store de sesión
ARTAGRAM-Infraestructure/  docker-compose.yml, Prometheus, Grafana (dashboards provisionados), este README
```

Los 3 repos se clonan como carpetas hermanas, al mismo nivel.

## Cómo levantar todo

Este proyecto vive en **3 repositorios separados** (`ARTAGRAM-BackEnd`, `ARTAGRAM-FrontEnd`,
`ARTAGRAM-Infraestructure`), conectados por imágenes Docker publicadas en GitHub Container
Registry, no por rutas de carpeta compartidas — ver [el apartado de más abajo](#cómo-están-conectados-los-3-repositorios)
para el detalle de esa conexión. Formas de levantar el stack:

### Opción A — Imágenes publicadas (como va a funcionar una vez separado en 3 repos)

```bash
cp .env.example .env     # completá GHCR_OWNER con tu usuario de GitHub
docker compose pull
docker compose up -d
```

### Opción B — Build local desde el código fuente de los 3 repos clonados como hermanos

```bash
docker compose -f docker-compose.yml -f docker-compose.dev.yml up --build -d
```

`docker-compose.dev.yml` pisa el `image:` del archivo base por un `build: context: ../ARTAGRAM-BackEnd`
/ `../ARTAGRAM-FrontEnd` — por eso necesita que los 3 repos (`ARTAGRAM-BackEnd`, `ARTAGRAM-FrontEnd`,
`ARTAGRAM-Infraestructure`) estén clonados como carpetas hermanas, todas al mismo nivel.

Cualquiera de las dos formas levanta Postgres, Redis, el backend (puerto `8080`), el
frontend servido por nginx (puerto `5173`), Prometheus (`9090`) y Grafana (`3001`, usuario
`admin`/`admin`, con acceso anónimo de solo lectura habilitado). Abrí `http://localhost:5173`.

Variables de entorno relevantes (todas con default de desarrollo, ver
`docker-compose.yml` en este repo y `application.yml` en ARTAGRAM-BackEnd):

| Variable | Uso | Default dev |
|---|---|---|
| `GHCR_OWNER` | tu usuario/org de GitHub — de ahí sale el nombre de las imágenes en `ghcr.io` (solo Opción A) | `tu-usuario-de-github` (placeholder, no existe) |
| `JWT_SECRET` | firma de los JWT (REST + STOMP) | secreto de desarrollo, **cambiar en producción** |
| `MEDIA_SIGNING_SECRET` | HMAC de las URLs firmadas de `/media/**` | secreto de desarrollo |
| `CORS_ORIGINS` | orígenes permitidos | `http://localhost:5173` |
| `APP_BASE_URL` | base para el enlace de verificación de correo | `http://localhost:5173` |
| `APP_API_BASE_URL` | base pública del backend (debe coincidir con el puerto real expuesto) | `http://localhost:8080` |
| `EMAIL_MODE` | `log` (loguea el enlace, sin SMTP real) o `smtp` | `log` |
| `VITE_API_BASE_URL` / `VITE_WS_URL` | a dónde apunta el frontend (se hornean en el build de la imagen) | `http://localhost:8080` / `ws://localhost:8080/ws` |

> Si corrés el backend en un puerto distinto de 8080 (por ejemplo porque ese puerto ya
> está ocupado localmente), acordate de exportar `APP_API_BASE_URL` con ese puerto —
> si no, las URLs firmadas de imágenes (`obraFinalUrl`, fotos de perfil) quedan apuntando
> al puerto equivocado y las imágenes no cargan (el resto de la app funciona igual).

### Poblar la base con datos de demostración

El backend trae un `DemoDataSeeder` (Java puro, `@Profile("seed")`) que crea usuarios,
comunidades, publicaciones con imágenes de ejemplo, likes, seguidores, chats y banners
pintados — para que la app no arranque vacía. Es idempotente (si ya corrió, no duplica
nada) y solo se activa con el profile `seed`:

```bash
# con Docker: exportá el profile antes de levantar el backend (o agregalo a tu .env)
SPRING_PROFILES_ACTIVE=seed docker compose -f docker-compose.yml -f docker-compose.dev.yml up --build -d

# corriendo el backend a mano (desde ARTAGRAM-BackEnd)
mvn spring-boot:run -Dspring-boot.run.profiles=seed
```

Uno de los usuarios sembrados (`prueba.artagram@correo.com`) queda promovido a rol
`ADMIN`, para poder entrar al panel de administración sin tocar la base a mano.

### Opción C — correr cada parte a mano, sin Docker (para desarrollo)

```bash
# Postgres y Redis (podés usar solo esos dos servicios del compose), desde ARTAGRAM-Infraestructure
docker compose up postgres redis

# Backend, desde ARTAGRAM-BackEnd (otra terminal)
mvn spring-boot:run

# Frontend, desde ARTAGRAM-FrontEnd (otra terminal)
npm install
npm run dev   # usa .env (VITE_API_BASE_URL/VITE_WS_URL apuntan a :8080 por defecto);
              # si tu backend corre en otro puerto, creá un .env.local con esas dos
              # variables apuntando ahí — Vite lo prioriza sobre .env
```

## Cómo están conectados los 3 repositorios

La separación en `ARTAGRAM-BackEnd`, `ARTAGRAM-FrontEnd` y `ARTAGRAM-Infraestructure`
ya está hecha. La "conexión" entre ellos no es por ruta de carpeta compartida sino por
imágenes Docker: cada uno de los repos de backend y frontend trae su propio workflow de
GitHub Actions (`.github/workflows/docker-publish.yml`) que compila, corre las pruebas
(solo el backend) y publica la imagen en `ghcr.io/<tu-usuario>/artagram-backend` /
`artagram-frontend` en cada push a `main` o `master`. `docker-compose.yml` en este repo
ya apunta a esas imágenes por nombre — no necesita ver el código fuente de los otros dos
repos para nada.

Para que la Opción A (imágenes publicadas) funcione en una máquina nueva:

1. En cada uno de `ARTAGRAM-BackEnd` y `ARTAGRAM-FrontEnd`, el primer push a `main`/`master`
   dispara el workflow y publica la imagen en GHCR.
2. **Hacé público el paquete publicado** (la primera vez que corre el workflow, GHCR lo
   crea como privado): en GitHub → tu perfil → *Packages* → `artagram-backend` →
   *Package settings* → *Change visibility* → *Public*. Repetí para `artagram-frontend`.
   Si preferís dejarlos privados, `docker login ghcr.io` con un Personal Access Token
   (scope `read:packages`) antes del `docker compose pull`.
3. **En este repo (`ARTAGRAM-Infraestructure`)**, copiá `.env.example` a `.env` y poné tu
   usuario de GitHub en `GHCR_OWNER`. A partir de ahí, `docker compose pull && docker compose up -d`
   en *cualquier máquina* levanta el stack completo sin tocar código fuente de los otros
   dos repos — esa es toda la conexión.

Si más adelante cambiás algo del backend o del frontend, el flujo normal es: commit →
push a tu repo → el workflow republica `:latest` solo → en la máquina donde corre infra,
`docker compose pull && docker compose up -d` trae la versión nueva.

## Cómo correr las pruebas

```bash
# desde ARTAGRAM-BackEnd
mvn test      # unitarias (JUnit5 + Mockito), no necesitan Docker
mvn verify    # + integración: Testcontainers levanta Postgres y Redis reales,
              # y las pruebas de tiempo real usan un cliente STOMP real (no mocks)
```

Reporte de cobertura JaCoCo: `target/site/jacoco/index.html` (dentro de ARTAGRAM-BackEnd) tras `mvn verify`.
Análisis estático: `mvn spotbugs:check` (ver el detalle en
[`docs/gherkin-coverage.md`](docs/gherkin-coverage.md#cobertura-y-análisis-estático) de ese repo).
Documentación interactiva de la API (Swagger UI), con el backend corriendo:
`http://localhost:8080/swagger-ui.html`.

```bash
# desde ARTAGRAM-FrontEnd
npm run build   # tsc -b && vite build — falla si hay errores de tipos
```

## Cómo probar a mano los flujos de tiempo real

Con el stack levantado (Docker Compose o a mano):

1. **Registro y verificación**: registrate en `/registro`. Como no hay SMTP configurado
   por defecto (`EMAIL_MODE=log`), el enlace de verificación queda en los **logs del
   backend** (buscá `[EMAIL-DEV] Verificacion para ...`) en vez de llegar por correo.
   Copiá esa URL en el navegador para confirmar la cuenta.
2. **Lienzo colaborativo (HU-12/13/15)**: creá una comunidad, copiá su ID (se puede ver
   en la tarjeta o en la URL de `/comunidades/:id`), y con una **segunda cuenta en otro
   navegador o una ventana de incógnito** (los navegadores comparten `localStorage` entre
   pestañas del mismo perfil, así que dos pestañas normales del mismo navegador van a
   compartir la sesión — para simular dos usuarios de verdad hace falta un segundo
   perfil/navegador) unite con "Unirme por ID de comunidad" y entrá al mismo lienzo.
   Dibujá desde ambos lados: los trazos aparecen en tiempo real en la otra pantalla con
   el mismo pincel/color/grosor. Recargá una de las dos pestañas a la mitad de un trazo
   ajeno: al volver a cargar, el lienzo se reconstruye completo desde el snapshot. Para
   HU-15, activá "Editar un trazo existente" y hacé clic cerca de un trazo — queda
   bloqueado (el otro usuario ve un aviso "fulano está editando"); si el otro intenta
   tomarlo mientras tanto, el backend lo rechaza.
3. **Banner mural (HU-16)**: desde la comunidad, entrá a "Banner mural" con las dos
   cuentas y pinten en regiones distintas (o la misma) del mural de 1000×1000 — cada
   región tiene su propio lock, así que dos personas en regiones distintas nunca se
   bloquean entre sí.
4. **Modo fiesta con dos personas a la vez (HU-18/19/20)** — el paso a paso completo:
   1. Abrí dos navegadores *distintos* (o uno normal + uno de incógnito — ver la nota
      de arriba sobre por qué dos pestañas normales no alcanzan) en `http://localhost:5173`.
   2. En el navegador A, registrá/logueá la cuenta 1 y andá a "Modo fiesta" → "Crear
      sala". Copiá el **código de 6 caracteres** que aparece arriba a la izquierda
      (ej. `2XCQPV`).
   3. En el navegador B, registrá/logueá una cuenta 2 distinta, andá a "Modo fiesta" →
      pegá ese código en "Unirme con un código" → "Entrar a la sala". Deberías ver
      "2 en la sala" en ambos navegadores (puede tardar unos segundos en refrescarse).
   4. En el navegador A (el anfitrión), presioná "Iniciar ronda". **En los dos
      navegadores a la vez** debería aparecer el mismo pincel/color forzado arriba a la
      derecha y el mismo reloj corriendo en sincronía (con 1-2 décimas de diferencia
      como mucho, ya que el servidor es la única fuente de verdad).
   5. Dibujá desde el navegador A: el trazo tiene que aparecer casi al instante en el
      navegador B, con el mismo color/pincel — así confirmás que el WebSocket
      (`/topic/fiesta/{id}`) está entregando en ambas direcciones.
   6. Probá "Tomar el borrador" desde los dos navegadores casi al mismo tiempo: solo uno
      lo consigue (queda resaltado "Soltar borrador" en ese lado), y el otro ve
      "Borrador: <el que lo tiene>" — eso confirma la exclusión mutua (HU-20).
   7. Dejá que se acabe el tiempo: la ronda se congela igual en ambas pantallas al mismo
      momento (el servidor la corta, no cada cliente por su cuenta).
5. **Chat personal y de comunidad (HU-21/22)**: desde "Mensajes", buscá a la otra cuenta
   por su nombre de artista y escribile — si esa cuenta no está conectada en ese momento,
   el mensaje queda guardado y le llega apenas se conecta. El chat de la comunidad
   (dentro del detalle de la comunidad) es visible para todos los miembros conectados.
6. **Feed y likes (HU-05/06/07)**: publicá una obra (imagen final obligatoria, hasta 10
   imágenes de proceso opcionales) desde "Feed" → "Publicar", y dale like desde la otra
   cuenta — el contador se actualiza al instante. El ícono de basurero solo aparece en tus
   propias publicaciones — con la otra cuenta, intentá borrar la publicación de la primera
   directo por API (`DELETE /api/publicaciones/{id}`) y confirmá que el backend responde
   403 `SOLO_AUTOR` aunque el botón no esté visible en esa cuenta.
7. **Panel de administración (HU-38)**: no hay forma de auto-promoverse a admin desde la
   UI (a propósito). Si levantaste el backend con el profile `seed` (ver [Poblar la base
   con datos de demostración](#poblar-la-base-con-datos-de-demostración)), la cuenta
   `prueba.artagram@correo.com` / `clavesegura123` ya es `ADMIN` — iniciá sesión con esa
   y entrá a `/admin`. Sin seed, promové un usuario a mano en la base:
   ```sql
   UPDATE usuarios SET rol_global = 'ADMIN' WHERE correo = 'tu-correo@ejemplo.com';
   ```
   Volvé a iniciar sesión con esa cuenta y entrá a `/admin` — vas a ver los 4 KPIs de
   negocio con su fórmula y tendencia, y los enlaces a los paneles de Grafana. Con una
   cuenta sin rol ADMIN, ni siquiera aparece el link "Admin" del menú, y si se entra
   igual por URL directa el backend rechaza la petición con 403 (`SIN_PERMISO`) y el
   intento queda registrado en los logs con nivel `WARN`.
8. **Métricas técnicas**: `http://localhost:8080/actuator/prometheus` expone, entre
   otras, `trazo_latencia_ms_seconds`, `trazos_por_segundo_total`,
   `bloqueos_negados_total`, `borrador_disputas_total`, `ws_sesiones_activas` y
   `http_errores_5xx_total`. El dashboard técnico de Grafana (`localhost:3001`) ya viene
   provisionado con paneles para todas ellas. El **panel de negocio** (los 4 KPIs de
   HU-37, `localhost:3001/d/artagram-negocio`) usa las mismas métricas pero expuestas
   como gauges (`kpi_tasa_colaboracion`, `kpi_duracion_sesion_minutos`,
   `kpi_retencion_7_dias`, `kpi_conversion_premium`) — mismo datasource de Prometheus,
   sin necesitar un datasource de Postgres aparte en Grafana.

## Despliegue en producción

Todo lo de arriba (Docker Compose, Prometheus, Grafana) es para **desarrollo y demo
local**. Para desplegar de verdad, cada pieza va a un servicio distinto — no hay un
único "deploy de infra" porque ya no comparten host:

- **Backend → Azure App Service** (contenedor Linux, tirando de la imagen que ya publica
  el CI en `ghcr.io`), con Azure Database for PostgreSQL y Azure Cache for Redis como
  dependencias administradas, y Azure Blob Storage para los archivos subidos
  (`STORAGE_PROVIDER=blob` — sin esto, el disco del contenedor no persiste entre
  reinicios). Paso a paso completo y variables de entorno en el
  [README de `ARTAGRAM-BackEnd`](https://github.com/Cam1lo27/ARTAGRAM-BackEnd#despliegue-en-azure)
  — ahí también está por qué hay que prender "Web sockets" a mano en Azure (viene
  apagado por defecto y sin eso no funciona nada de tiempo real).
- **Frontend → Vercel**, construyendo directo desde el código fuente (no necesita la
  imagen Docker de este repo). Paso a paso, `vercel.json` y variables de entorno en el
  [README de `ARTAGRAM-FrontEnd`](https://github.com/Cam1lo27/ARTAGRAM-FrontEnd#despliegue-en-vercel).
- **Observabilidad**: Prometheus/Grafana tal como están acá son para correr al lado del
  backend (scrapean `backend:8080` por nombre de servicio de Docker Compose). Llevarlos a
  producción junto al App Service de Azure — o migrar a Azure Monitor/Grafana Cloud —
  quedó fuera de alcance de esta entrega; el backend ya expone
  `/actuator/prometheus` listo para que cualquiera de las dos opciones lo consuma el día
  que se decida.

Lo importante para no perderse: **backend y frontend se despliegan y versionan de forma
completamente independiente** (cada uno con su propio CI/CD, ver
["Cómo están conectados los 3 repositorios"](#cómo-están-conectados-los-3-repositorios)) —
desplegar uno no requiere redesplegar el otro, solo que `CORS_ORIGINS` (backend) y
`VITE_API_BASE_URL`/`VITE_WS_URL` (frontend) apunten correctamente el uno al otro.

## Qué queda fuera del alcance (a propósito)

Todo lo que el story map marca como Release 2/3/4 — login con Google, notificaciones,
promociones/monetización avanzada, funciones con IA, seguir/dejar de seguir completo
(HU-04, solo hay el soporte mínimo que usa el feed para sugerir artistas), y cursores en
vivo de otros usuarios sobre el lienzo (HU-14) — no está implementado. Se evaluó agregar
cursores en vivo como mejora estética ya que el pedido de diseño lo sugiere como
micro-interacción deseable, pero como HU-14 es explícitamente Release 2 se decidió no
construir infraestructura nueva de backend para una historia fuera del MVP sin
confirmarlo antes.
