# Red de la biblioteca · L9

## AWS · VPC 10.0.0.0/16 · us-east-1

| Subred | Rango | Zona | Ruta 0.0.0.0/0 |
|---|---|---|---|
| pública | 10.0.1.0/24 | us-east-1a | Internet Gateway |
| aplicación | 10.0.11.0/24 | us-east-1a | interfaz de red de `nat` |
| datos-a | 10.0.21.0/24 | us-east-1a | interfaz de red de `nat` |
| datos-b | 10.0.22.0/24 | us-east-1b | interfaz de red de `nat` |

| Máquina | Subred | IP privada | Tamaño | Lo que corre | Por qué ese tamaño |
|---|---|---|---|---|---|
| `gw-aws` | pública | 10.0.1.20, con Elastic IP | t3.small | Caddy con HTTPS y `biblioteca-gateway` | Medido: gateway 14 MiB + Caddy y el SO; 2 GiB dejan holgura para TLS y picos de tráfico. |
| `nat` | pública | 10.0.1.10 | t3.nano | NAT instance de las subredes privadas | No corre contenedores: solo reenvía paquetes; 0,5 GiB alcanzan para iptables. |
| `servicios` | aplicación | 10.0.11.10 | t3.medium | `biblioteca-libros`, `biblioteca-prestamos` y `biblioteca-eventos` | Medido: libros 11 + prestamos 57 + eventos 67 = 135 MiB; con Docker y el SO (~0,6 GiB) cabe en 4 GiB y deja holgura para picos. |
| `gestion` | aplicación | 10.0.11.20 | t3.medium | `biblioteca-admin`, Kafka UI, `biblioteca-simulador`, runner `aws` (Prometheus y Grafana, opcionales) | Medido: admin 43 + Kafka UI 306 = 349 MiB; el simulador y Prometheus/Grafana opcionales suman; 4 GiB cubren el conjunto. |
| `mq-1` | datos-a | 10.0.21.11 | t3.small | controller 1, broker 4 y `rabbit1` | Medido: controller 214 + broker 332 + rabbit1 120 = 666 MiB; 2 GiB cubren el SO y el heap de Java. |
| `mq-2` | datos-b | 10.0.22.12 | t3.small | controller 2, broker 5 y `rabbit2` | Medido: controller 120 + broker 354 + rabbit2 114 = 588 MiB; 2 GiB cubren el SO y el heap de Java. |
| `mq-3` | datos-a | 10.0.21.13 | t3.small | controller 3 y broker 6 | Medido: controller 158 + broker 491 = 649 MiB; sin RabbitMQ, 2 GiB sobran. |
| `datos` | datos-a | 10.0.21.30 | t3.small | Postgres 18 | Medido: postgres 28 MiB en reposo; el resto es caché de disco y margen para la carga de la E3. |

| Grupo de reglas | Puerto | Origen | Para qué |
|---|---|---|---|
| `sg-gw` | 80 | `0.0.0.0/0` | Desafío de Let's Encrypt y redirección a HTTPS |
| `sg-gw` | 443 | pendiente E2 | La API, solo para el BFF |
| `sg-servicios` | 3001, 3002 y 3010 | `sg-gw` y `sg-gestion` | El gateway llega a `libros` y `prestamos`; el administrador revisa `eventos` |
| `sg-mq` | 9092 | `sg-servicios`, `sg-gestion` y `sg-mq` | Clientes de Kafka y réplica entre brokers |
| `sg-mq` | 9093 | `sg-mq` | Quórum KRaft |
| `sg-mq` | 5672 | `sg-servicios` y `sg-gestion` | AMQP de RabbitMQ |
| `sg-mq` | 15672 | `sg-gestion` | Administración de RabbitMQ |
| `sg-mq` | 4369 y 25672 | `sg-mq` | Clúster de RabbitMQ |
| `sg-datos` | 5432 | `sg-servicios` y `sg-gestion` | Postgres |
| `sg-nat` | todos | `sg-servicios`, `sg-gestion`, `sg-mq` y `sg-datos` | Salida a internet de las subredes privadas |
| `sg-gestion` | ninguno | | Se entra por Session Manager |

## Azure · VNet 10.1.0.0/16 · northcentralus (por confirmar; pendiente E2)

| Subred | Rango | Salida a internet |
|---|---|---|
| pública | 10.1.1.0/24 | directa, por la IP pública de `gw-azure` |
| privada | 10.1.2.0/24 | por `gw-azure` (tabla de rutas hacia 10.1.1.4); creada con `--default-outbound false` |

| Máquina | Subred | IP privada | Tamaño | Lo que corre | Por qué ese tamaño |
|---|---|---|---|---|---|
| `gw-azure` | pública | 10.1.1.4, con IP pública estática y etiqueta pendiente E2 | B2ats_v2 | Caddy, `biblioteca-web`, `biblioteca-gateway` y NAT de la subred privada | Medido: web 2 + gateway 14 = 16 MiB; con Caddy, el SO y el NAT, 1 GiB alcanza y es el tamaño más barato. |
| `bff` | privada | 10.1.2.4, sin IP pública | B2als_v2 | `biblioteca-bff` y el runner `azure` | Medido: bff 42 MiB; 4 GiB dejan holgura para el runner y para picos de la E3. |

| NSG | Puerto | Origen | Para qué |
|---|---|---|---|
| `gw-azure` | 443 y 80 | Internet | La web y el desafío de Let's Encrypt |
| `gw-azure` | 22 | pendiente E2 | SSH |
| `gw-azure` | todos | 10.1.2.0/24 | El NAT de `bff` |
| `bff` | todos | 10.1.0.0/16 | Solo desde la VNet |
