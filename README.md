# biblioteca-plataforma

Levanta el sistema entero —Postgres, RabbitMQ en clúster, y los siete servicios: `biblioteca-eventos`,
`biblioteca-libros`, `biblioteca-prestamos`, `biblioteca-admin`, `biblioteca-bff`, `biblioteca-gateway` y
`biblioteca-web`— con un solo comando. Este repositorio no tiene código de aplicación: solo `compose.yml`,
`.env.example`, `docs/` y `rabbitmq/`.

| Servicio | Repositorio | Puerto |
|---|---|---|
| `web` | `biblioteca-web` | 4200 |
| `gateway` | `biblioteca-gateway` | 8080 |
| `bff` | `biblioteca-bff` | 3000 |
| `libros` | `biblioteca-libros` | 3001 |
| `prestamos` | `biblioteca-prestamos` | 3002 |
| `eventos` | `biblioteca-eventos` | 3010 |
| `admin` | `biblioteca-admin` | 3020 |
| `postgres` / `rabbit1`, `rabbit2` | (imágenes oficiales) | 5432 / 5672, 5673 y 15672, 15673 |

## Cómo clonar los ocho y levantar

Los ocho repositorios van **uno al lado del otro**, en la misma carpeta, con el nombre **sin** el sufijo `-l9`
(el `compose.yml` busca `../biblioteca-<pieza>`). La lista con los enlaces está en `docs/repositorios.md`. Tras hacer
Fork de cada uno, y parado en la carpeta donde quieras tenerlos, los mismos comandos sirven en Windows
(PowerShell), macOS y Linux:

```bash
cd $HOME/DSY1107
mkdir l9
cd l9
git clone https://github.com/TU_USUARIO/biblioteca-plataforma-l9.git biblioteca-plataforma
git clone https://github.com/TU_USUARIO/biblioteca-web-l9.git biblioteca-web
git clone https://github.com/TU_USUARIO/biblioteca-gateway-l9.git biblioteca-gateway
git clone https://github.com/TU_USUARIO/biblioteca-bff-l9.git biblioteca-bff
git clone https://github.com/TU_USUARIO/biblioteca-libros-l9.git biblioteca-libros
git clone https://github.com/TU_USUARIO/biblioteca-prestamos-l9.git biblioteca-prestamos
git clone https://github.com/TU_USUARIO/biblioteca-eventos-l9.git biblioteca-eventos
git clone https://github.com/TU_USUARIO/biblioteca-admin-l9.git biblioteca-admin
```

Cada repositorio con variables trae un `.env.example`, con los nombres y sin los valores. Cópialo a `.env` y
rellénalo (Windows: `Copy-Item .env.example .env`; macOS y Linux: `cp .env.example .env`) en **estos seis**:
`biblioteca-plataforma`, `biblioteca-gateway`, `biblioteca-bff`, `biblioteca-prestamos`, `biblioteca-eventos` y
`biblioteca-admin`. `biblioteca-libros` y `biblioteca-web` no leen ningún `.env` al levantar con Docker.

Después, desde `biblioteca-plataforma`:

```bash
bash levantar.sh
```

Termina cuando todos los servicios están `healthy` (o arriba, los que no tienen comprobación de salud).
`docker compose ps` los lista. Para bajarlo sin borrar los datos: `docker compose down`.

Si cambias un repositorio y quieres reconstruir solo ese servicio: `docker compose up -d --build <servicio>`
(`gateway`, `bff`, `prestamos`, ...).

## El clúster (L8)

```
   biblioteca-admin :3020 ──AMQP──> :5672  rabbit1 ═══ rabbit2  :5673
                          ──HTTP──> :15672                      :15673
   biblioteca-eventos :3010 ──────> :5672 rabbit1,  :5432 Postgres
```

Dos nodos, `rabbit@rabbit1` y `rabbit@rabbit2`, con la misma cookie de Erlang (`RABBITMQ_COOKIE` del
`.env`). El clúster lo arma `rabbitmq/cluster.conf`, montado en los dos nodos: en una máquina limpia
un solo `docker compose up -d --wait` deja los dos unidos. Las seis colas son quorum.

Procedimiento de levantamiento desde un clon limpio, en `biblioteca-plataforma`:

```bash
docker compose up -d --build --wait
docker compose exec rabbit1 rabbitmqctl cluster_status
docker compose exec rabbit1 rabbitmqctl import_definitions /etc/rabbitmq/politicas.json
```

`cluster_status` tiene que traer los dos nodos en `Running Nodes`. Las políticas viven en
`rabbitmq/politicas.json` y están explicadas en `docs/politicas.md`; las métricas que se vigilan, en
`docs/metricas.md`.

Si se recrea `rabbit1` (por ejemplo al cambiar sus volúmenes), el worker y `prestamos` quedan
colgados de la conexión vieja: `docker compose restart eventos prestamos`.

## La API de administración (`biblioteca-admin`, puerto 3020)

Todas exigen un access token del grupo `bibliotecarios`.

| Verbo y ruta | Qué hace |
|---|---|
| `GET /v1/admin/colas` | Lista las colas (API HTTP de management) |
| `POST /v1/admin/colas` | Crea una cola; quorum por omisión |
| `DELETE /v1/admin/colas/:nombre` | Borra una cola |
| `GET /v1/admin/exchanges` | Lista los exchanges |
| `POST /v1/admin/exchanges` | Crea un exchange `direct`, `topic`, `fanout` o `headers` |
| `DELETE /v1/admin/exchanges/:nombre` | Borra un exchange |
| `POST /v1/admin/bindings` | Crea un binding cola + exchange + routing key |
| `DELETE /v1/admin/bindings` | Borra un binding, con los tres datos en el cuerpo |

## Retención de las DLQ

Elegimos `reject-publish` también en las DLQ: si una DLQ llega a mil mensajes el sistema lleva rato
roto, y preferimos que se note fuerte. Lo que se pierde: el mensaje que iba a morir número 1001 no
llega a la DLQ y no queda en `mensajes_muertos`. Con `drop-head` se perderían en silencio los más
viejos.

## Mensaje envenenado

Desde `biblioteca-plataforma`, con el worker corriendo:

```bash
docker compose run --rm --no-deps eventos node herramientas/emitir.mjs 1 --roto
```

Qué se tiene que ver: dos `ERROR ... hacia la DLQ` en `docker compose logs eventos` (auditoria y
notificaciones) y `docker compose exec rabbit1 rabbitmqctl list_queues name messages` con
`auditoria.dlq 1` y `notificaciones.dlq 1`.
