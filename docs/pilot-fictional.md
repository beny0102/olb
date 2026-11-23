# Piloto ficticio de Fase 1

**Tipo:** simulacro de mesa con datos sintéticos. No es un piloto de liga real,
no sustituye una prueba de carga y no crea consentimientos legales.

**Ejecución:** el guion se ejecutó el 2026-11-23 sobre `olb_pilot_auto`. Resultado
y evidencia en [pilot-run-2026-11-23.md](pilot-run-2026-11-23.md): P-01 a P-11
pasan; P-12 queda parcial porque no se ensayó el rollback de código.

El objetivo es recorrer el criterio de salida de Fase 1 sin depender de una liga:
un juego de nueve entradas anotado desde el teléfono, sincronizado desde el
navegador offline y seguido en un segundo dispositivo. Todos los equipos,
jugadores, tutores y correos son ficticios. No se usa PII real.

## Elenco y datos

| Rol | Quién lo opera | Datos |
| --- | --- | --- |
| Administrador de liga | Operador A | Cuenta demo `admin@liga-metropolitana.olb` |
| Anotador de Tiburones | Operador A, teléfono | Cuenta demo de Tiburones |
| Espectador | Operador B, segunda pantalla | Sin sesión |
| Verificador | Operador A | Cuenta demo de verificador |
| Liga | — | `Liga Metropolitana` sembrada por el repo |

Las credenciales demo están en la tabla de `olb-api/README.md`. No se reutilizan
fuera de una base de datos desechable.

El juego de liga sembrado es **Tiburones visita a Faros**, id
`0192f3d0-0000-7000-8000-000000000201`, programado en la temporada activa. La
fecha del seed es ficticia; si se prefiere un juego nuevo, el administrador puede
crearlo desde `/admin` y el resto del guion no cambia.

## Entorno aislado

No usar la base `olb` del demo para el piloto: el log es append-only y el ensayo
no debe contaminar los datos de evaluación.

```bash
docker exec olb-postgres createdb -U postgres olb_pilot

cd olb-api
DATABASE_URL=postgres://postgres:olb@127.0.0.1:5432/olb_pilot \
  SEED=1 PUBLIC_HOST=127.0.0.1 API_PORT=3601 WEB_PORT=3600 \
  ./deploy/up.sh
```

Si la base ya existe, usar otro nombre (`olb_pilot_2`) en lugar de borrarla.
Al terminar, guardar evidencia y solo entonces:

```bash
cd olb-api && ./deploy/down.sh
docker exec olb-postgres dropdb -U postgres olb_pilot
```

## Ejecución completada

El 2026-11-23 se ejecutó el guion completo sobre una base desechable, con un
teléfono Android y una segunda pantalla. El informe con los IDs, eventos,
hashes y reservas está en [pilot-run-2026-11-23.md](pilot-run-2026-11-23.md).
Los criterios P-01 a P-11 pasan; P-12 queda parcial por el rollback de código.

## Guion

### 0. Preparación

1. Confirmar `/health` en `http://127.0.0.1:3601/health` y `/` en
   `http://127.0.0.1:3600`.
2. Tomar un backup de `olb_pilot` con
   `POSTGRES_DB=olb_pilot BACKUP_CONTAINER=olb-postgres ./deploy/backup.sh`.
3. Abrir dos perfiles de navegador distintos: anotador y espectador. No mezclar
   sesiones en el mismo perfil.

### 1. Privacidad y consentimiento

1. Abrir `/teams/tiburones` sin sesión. Confirmar que la plantilla muestra
   `#jersey` o posición, nunca nombres completos.
2. Entrar como administrador en `/admin`, buscar un jugador ficticio y autorizar
   su nombre con tutor `Tutor Sintético` y parentesco `demo`.
3. Recargar `/teams/tiburones`: solo ese jugador muestra su nombre.
4. Revocar el consentimiento y confirmar que vuelve a `#jersey`.
5. Dejar constancia de que el consentimiento es un dato de prueba, no una
   autorización real.

### 2. Anotación online y offline

1. En el teléfono, entrar a `/score` e iniciar sesión como Tiburones.
2. Seleccionar el juego programado Tiburones visita a Faros y prepararlo.
3. Cargar alineaciones completas de ambos equipos, con pitcher designado.
4. Iniciar el juego y anotar media entrada con red.
5. En el espectador, abrir `/games/<id>`: el marcador debe actualizarse en pocos
   segundos.
6. Activar modo avión o `Network → Offline` en DevTools **después** de cargar la
   pantalla de anotación.
7. Anotar el resto de las nueve entradas sin red, incluyendo:
   - una sustitución;
   - un corredor emergente o cortesía;
   - una nota;
   - un evento corregido mediante enmienda o `revisionOf`.
8. Recargar la pantalla de anotación una vez sin red y confirmar que el juego
   local sigue disponible.
9. Restaurar la red y sincronizar. No debe haber eventos duplicados.

### 3. Verificación y cierre

1. Como verificador, abrir `/score/<id>/verify`.
2. Revisar la enmienda y los eventos dudosos.
3. Cerrar el juego con el marcador que resulte del fold, no con un número
   escrito a mano.
4. Abrir `/games/<id>` y `/stats`: line score, box score y standings deben
   coincidir con el log.
5. Exportar el bundle local desde `/score` e importarlo en un perfil limpio.
   Confirmar que el juego se puede abrir sin la API.

### 4. Operación y rollback

1. Restaurar el backup en una base desechable y comparar conteos de juegos,
   jugadores, eventos y migraciones.
2. Ejecutar `./deploy/down.sh`.
3. Ensayar el rollback del checklist de release: tag anterior, restaurar dump y
   volver a arrancar con `SEED=0`.
4. No borrar el dump ni las capturas hasta cerrar el informe.

## Criterios de aceptación

| ID | Criterio | Evidencia |
| --- | --- | --- |
| P-01 | Stack levanta con base aislada y seed | log de `up.sh`, `/health` |
| P-02 | Nombres privados por defecto | captura de `/teams/tiburones` |
| P-03 | Consentimiento muestra y revoca el nombre | captura antes/después y fila en `player_consents` |
| P-04 | Juego de liga visible para el scorer de Tiburones | captura de `/score` |
| P-05 | Alineaciones completas y juego iniciado | `GET /games/:id/state` |
| P-06 | Anotación offline durante al menos media entrada | captura de cola local y log del service worker |
| P-07 | Sincronización sin duplicados | `lastSeq`, conteo de eventos, `409` ausente |
| P-08 | Espectador sigue el juego en segunda pantalla | dos capturas con el mismo marcador |
| P-09 | Enmienda y sustitución reflejadas en el fold | `GET /events`, estado final |
| P-10 | Cierre con marcador derivado del log | `GET /boxscore`, `/stats`, standings |
| P-11 | Bundle exportado e importado sin API | captura del juego local |
| P-12 | Backup restaurado y rollback ensayado | conteos y log de restauración |

Un fallo en P-02, P-06, P-07 o P-10 bloquea el piloto. Los demás se registran y
se corrigen antes de un piloto con una liga real.

## Informe

Guardar en una carpeta fuera del repositorio:

- fecha, commit y tag de API/web/contrato;
- URL y puertos usados;
- navegador, sistema y dispositivo;
- capturas de P-02 a P-11;
- salida de `GET /state`, `GET /boxscore` y `GET /events`;
- dump y hash del backup;
- defectos encontrados, con `pasos / esperado / obtenido`;
- decisión: pasa, pasa con reservas o no pasa.

## Límites

- El consentimiento sintético no tiene validez legal ni debe copiarse a producción.
- No se prueban cámaras, video, ni trayectorias reales.
- Una sola réplica de API en este guion: SSE y el throttle de login viven en
  proceso. El paquete escalado tiene su propio smoke; repetir el piloto contra
  dos réplicas queda pendiente.
- El objetivo es functional y operativo; carga, latencia multi-dispositivo y
  seguridad de red se prueban aparte.
