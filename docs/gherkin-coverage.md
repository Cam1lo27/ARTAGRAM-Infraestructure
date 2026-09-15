# Trazabilidad HU → escenario → prueba automatizada

Alcance: las 21 historias *Must have* del Release 1 (MVP) listadas en el story map.
Cada fila es un escenario Gherkin del documento de historias de usuario, mapeado a la
prueba automatizada (unitaria o de integración) que lo verifica. "Tipo" indica si la
prueba corre con `mvn test` (unit, Surefire) o `mvn verify` (IT, Failsafe + Testcontainers
con Postgres y Redis reales, más un cliente STOMP real para los canales de tiempo real).

| HU | Historia | Escenario cubierto | Prueba | Tipo |
|----|----------|---------------------|--------|------|
| HU-01 | Registro con verificación de correo | Registro válido queda pendiente de verificar y el login funciona tras verificar | `AuthServiceTest.registroExitosoQuedaPendienteDeVerificarYEnviaCorreo`, `AuthControllerIT.registroQueSalebienQuedaPendienteDeVerificarYPermiteLoginTrasVerificar` | unit + IT |
| HU-01 | Registro con verificación de correo | El correo ya usado no permite crear otra cuenta | `AuthServiceTest.correoYaUsadoNoCreaLaCuenta`, `AuthControllerIT.elCorreoYaEstaUsadoNoCreaLaCuenta` | unit + IT |
| HU-01 | Registro con verificación de correo | El nombre de artista ocupado se marca y sugiere alternativas libres | `AuthServiceTest.nombreDeArtistaOcupadoSugiereTresAlternativasLibres`, `AuthControllerIT.elNombreDeArtistaOcupadoSeMarcaYSugiereAlternativas` | unit + IT |
| HU-01 | Registro con verificación de correo | Login rechaza credenciales inválidas o cuenta no verificada | `AuthServiceTest.loginRechazaCredencialesInvalidas`, `AuthServiceTest.loginRechazaCuentaNoVerificada` | unit |
| HU-03 | Perfil público del artista | Guardar cambios de perfil (bio, enlaces, obras destacadas) los deja visibles de inmediato | `PerfilControllerIT.guardoCambiosDePerfilYQuedanVisiblesDeInmediato` | IT |
| HU-03 | Perfil público del artista | Alguien sin cuenta visita un perfil y solo ve lo público | `PerfilControllerIT.alguienVisitaMiPerfilSinCuentaVeSoloLoPublico` | IT |
| HU-03 | Perfil público del artista | Un perfil que no existe responde 404 | `PerfilControllerIT.perfilQueNoExisteDaNotFound` | IT |
| HU-05 | Publicar una obra con su proceso | Publicación completa queda con su línea de proceso en el orden subido | `PublicacionIT.publicacionCompletaQuedaConSuLineaDeProcesoEnOrden` | IT |
| HU-05 | Publicar una obra con su proceso | Un archivo con formato no permitido (validado por magic bytes) no se sube | `PublicacionIT.archivoConFormatoNoPermitidoNoSeSube` | IT |
| HU-05 | Publicar una obra con su proceso | Si falta la obra final no se deja publicar | `PublicacionIT.siSeOlvidaLaObraFinalNoDejaPublicar` | IT |
| HU-06 | Ver el feed de la comunidad | El feed trae lo más nuevo primero y pagina por cursor sin repetir ni saltar | `PublicacionIT.elFeedTraeLoMasNuevoPrimeroYPaginaSinRepetir` | IT |
| HU-06 | Ver el feed de la comunidad | Con el feed vacío se sugieren artistas para seguir | `PublicacionIT.feedVacioMeSugiereArtistas` | IT |
| HU-07 | Reaccionar con like | 200 usuarios dando like a la vez dejan el contador exacto en 200 (concurrencia real, contador atómico) | `LikeConcurrencyIT.doscientasPersonasDandoLikeALaVezDejanElContadorExactoEnDoscientos` | IT |
| HU-07 | Reaccionar con like | Volver a apretar el corazón lo quita y el contador nunca queda negativo | `LikeConcurrencyIT.volverAApretarElCorazonLoQuitaYElContadorNuncaQuedaNegativo` | IT |
| HU-10 | Crear y unirse a una comunidad | Crear una comunidad deja al creador como admin con lienzo y banner ya aprovisionados | `ComunidadControllerIT.creoLaComunidadYQuedoDeAdminConLienzoYBannerAprovisionados` | IT |
| HU-10 | Crear y unirse a una comunidad | El nombre de comunidad ya existente no deja crear otra | `ComunidadControllerIT.elNombreDeComunidadYaExisteNoDejaCrearOtra` | IT |
| HU-10 | Crear y unirse a una comunidad | Unirse a una comunidad pública deja como miembro | `ComunidadControllerIT.meUnoAUnaComunidadPublicaYQuedoComoMiembro` | IT |
| HU-11 | Roles y autorización por comunidad | Quien no es miembro no recibe ni un solo trazo del `/topic` de la comunidad (autorización en el handshake/SUBSCRIBE de STOMP, no solo en REST) | `CanvasWebSocketIT.quienNoEsMiembroNoRecibeNiUnSoloTrazo` | IT |
| HU-11 | Roles y autorización por comunidad | Alguien expulsado de la comunidad no puede escribir en su chat y nadie lo recibe | `ComunidadChatIT.alguienExpulsadoNoPuedeEscribirYNadieLoRecibe` | IT |
| HU-12 | Dibujar en tiempo real sobre el lienzo compartido | Mi trazo le llega a otro miembro conectado y queda con número de secuencia | `CanvasWebSocketIT.miTrazoLeLlegaAOtroMiembroYQuedaConSecuencia` | IT |
| HU-12 | Dibujar en tiempo real sobre el lienzo compartido | Alguien que llega tarde recibe todo lo ya dibujado, en orden (snapshot + replay sin perder trazos) | `CanvasWebSocketIT.alguienQueLlegaTardeRecibeTodoLoYaDibujadoEnOrden` | IT |
| HU-13 | Catálogo de pinceles con estilo propio | El trazo llega a otro miembro con el mismo tipo de pincel, color, grosor y opacidad (fidelidad de estilo) | `CanvasWebSocketIT.miTrazoLeLlegaAOtroMiembroYQuedaConSecuencia` | IT |
| HU-15 | Bloqueo de edición de un trazo | 6 usuarios intentan tomar el mismo trazo a la vez y solo uno gana el bloqueo | `TrazoLockIT.seisIntentanTomarElMismoTrazoYSoloUnoGana` | IT |
| HU-15 | Bloqueo de edición de un trazo | Solo el dueño del bloqueo puede editar; liberarlo lo deja disponible para otro | `TrazoLockIT.soloElDuenoDelBloqueoPuedeEditarYLiberarLoDejaDisponible` | IT |
| HU-16 | Banner mural comunitario por regiones | 30 personas pintando regiones distintas del banner a la vez no pierden ningún trazo | `BannerConcurrencyIT.treintaPersonasPintandoRegionesDistintasNoPierdenNada` | IT |
| HU-16 | Banner mural comunitario por regiones | Dos escrituras casi simultáneas en la misma región no se pierden (lock por región) | `BannerConcurrencyIT.dosEscriturasCasiSimultaneasEnLaMismaRegionNoSePierden` | IT |
| HU-16 | Banner mural comunitario por regiones | Quien se pasa del ritmo permitido (100 ops/s) se topa con el límite | `BannerConcurrencyIT.quienSePasaDeRitmoSeTopaConElLimite` | IT |
| HU-18 | Salas de modo fiesta con código de invitación | Crear una sala da un código y deja al creador como anfitrión | `PartyRoomIT.armoLaSalaYSaleUnCodigoQuedoComoAnfitrion` | IT |
| HU-18 | Salas de modo fiesta con código de invitación | Una sala llena no deja entrar a uno más | `PartyRoomIT.laSalaLlenaNoDejaEntrarAUnoMas` | IT |
| HU-19 | Rondas cronometradas con pincel único forzado | La ronda reparte el mismo pincel/color a todos y el reloj del servidor es igual para todos los participantes | `PartyRoomIT.laRondaReparteElMismoPincelYElRelojEsIgualParaTodos` | IT |
| HU-20 | El borrador como recurso disputado | 6 participantes pelean por el borrador a la vez y solo uno gana (compare-and-set) | `PartyEraserIT.seisParticipantesPeleanPorElBorradorYSoloUnoGana` | IT |
| HU-20 | El borrador como recurso disputado | Solo el dueño del borrador puede usarlo; al liberarlo queda disponible para otro | `PartyEraserIT.soloElDuenoDelBorradorPuedeUsarloYAlLiberarQuedaDisponible` | IT |
| HU-21 | Chat personal entre artistas | El mensaje llega de una cuando el destinatario está conectado | `ChatPersonalIT.elMensajeLlegaDeUnaCuandoElDestinatarioEstaConectado` | IT |
| HU-21 | Chat personal entre artistas | Si no está conectado el mensaje queda guardado y le llega al conectarse, en orden | `ChatPersonalIT.siNoEstaConectadoElMensajeQuedaGuardadoYLeLlegaAlConectarseEnOrden` | IT |
| HU-22 | Chat de la comunidad | Un mensaje le llega a todos los conectados y queda en el historial | `ComunidadChatIT.unMensajeLeLlegaATodosLosConectadosYQuedaEnElHistorial` | IT |
| HU-26 | Límites del plan gratuito | 5 solicitudes concurrentes de crear comunidad solo dejan completar el tope de 3 | `QuotaConcurrencyIT.cincoSolicitudesConcurrentesSoloUnaCompletaElTopeDeTres` | IT |
| HU-26 | Límites del plan gratuito | Llegar al tope de 3 hace que la cuarta se rechace | `QuotaConcurrencyIT.llegueAlTopeDeTresYLaCuartaSeRechaza` | IT |
| HU-26 | Límites del plan gratuito | Liberar cupo al eliminar una comunidad permite crear otra | `QuotaConcurrencyIT.liberoCupoAlEliminarYPuedoCrearOtra` | IT |
| HU-35 | Logs estructurados con trazabilidad | Se genera un traceId nuevo cuando el cliente no manda uno | `TraceIdFilterTest.generaUnTraceIdNuevoCuandoElClienteNoMandaUno` | unit |
| HU-35 | Logs estructurados con trazabilidad | Se reutiliza el traceId que manda el cliente (`X-Trace-Id`) | `TraceIdFilterTest.reutilizaElTraceIdQueMandaElCliente` | unit |
| HU-36 | Métricas técnicas expuestas | Las métricas técnicas (`/actuator/prometheus`) quedan expuestas para un admin | `ObservabilidadIT.lasMetricasTecnicasQuedanExpuestasParaUnAdmin` | IT |
| HU-37 | KPIs de negocio calculados | Entrar a un lienzo compartido sube la tasa de colaboración (KPI `ENTRO_COLABORATIVO`) | `KpiIT.entrarAUnLienzoCompartidoSubeLaTasaDeColaboracion` | IT |
| HU-37 | KPIs de negocio calculados | Los 4 KPIs de negocio responden con fórmula, periodo y conteo | `KpiIT.losCuatroKpisResponden` | IT |
| HU-38 | Panel administrativo restringido | Un usuario normal no entra al dashboard de admin y el intento queda registrado (log WARN) | `ObservabilidadIT.unUsuarioNormalNoEntraAlDashboardYQuedaRegistrado` | IT |
| HU-38 | Panel administrativo restringido | Un admin sí entra al dashboard | `ObservabilidadIT.unAdminSiEntraAlDashboard` | IT |

## Cómo correr todo

```bash
# desde el repo ARTAGRAM-BackEnd
mvn test      # unitarias (Surefire): AuthServiceTest, TraceIdFilterTest
mvn verify    # unitarias + de integración (Failsafe, Testcontainers Postgres+Redis reales)
```

`mvn verify` levanta un único Postgres y un único Redis (vía Testcontainers, compartidos
estáticamente entre todas las clases `*IT.java` para que la suite completa corra rápido)
y usa clientes STOMP reales (`StompTestClients`) para las pruebas de tiempo real — no hay
mocks de WebSocket ni de Redis en ninguna prueba de integración.

## Cobertura y análisis estático

- **JaCoCo**: `target/site/jacoco/index.html` tras `mvn verify` — cobertura de línea sobre
  los paquetes de dominio (auth, profile, community, canvas, party, chat, feed, billing,
  observability) está por encima del 70% acordado en el DoD para esta etapa.
- **SpotBugs**: `mvn spotbugs:check` (o `mvn verify`, corre en la fase `verify`). Con
  `effort=Max` y `threshold=Medium`, la suite reporta 4 hallazgos, todos de severidad
  **Medium** (ninguno High): 1 `CT_CONSTRUCTOR_THROW` (genérico, de bajo riesgo real —
  un constructor con validación) y 3 `NP_NULL_ON_SOME_PATH_FROM_RETURN_VALUE` revisados a
  mano y confirmados como falsos positivos (`Path.getParent()` sobre una ruta que siempre
  tiene ≥2 segmentos; `Map.merge` sobre `RMap`, cuyo contrato nunca invoca la función de
  remapeo con el valor viejo en `null`). El filtro `spotbugs-exclude.xml` suprime el ruido
  de `EI_EXPOSE_REP`/`EI_EXPOSE_REP2` (116 de los 120 hallazgos originales), esperable en
  cualquier código Spring/JPA con inyección de dependencias y entidades relacionadas, y
  documentado en el propio filtro.
