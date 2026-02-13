# Представление безопасность и инфраструктурное
Общая схема инфраструктуры

Система развёрнута в гибридной среде (2 ЦОД + облачные резервы) с геораспределённой архитектурой. Основные компоненты:
ЦОД‑1 ↔ ЦОД‑2 ↔ Облако (резерв)
│         │         └─> CDN для статического контента
├──> Kubernetes‑кластер (основной)
├──> PostgreSQL‑кластеры (Patroni)
├──> Kafka‑кластер
├──> Redis‑кластер
└──> Мониторинг‑стек (Grafana + Prometheus)
2. Физическая инфраструктура

2.1. ЦОД

ЦОД‑1 (основной):

3 ноды Kubernetes;

основной кластер БД;

Kafka‑брокеры.

ЦОД‑2 (резервный):

2 ноды Kubernetes;

реплики БД;

резервные Kafka‑брокеры.

Облако (AWS/GCP/Azure):

Резервные копии БД;

CDN (Cloudflare/AWS CloudFront);

Disaster Recovery‑ноды.

2.2. Сетевая связность

VPN‑туннели между ЦОД.

Балансировка трафика через HAProxy или Nginx.

DNS‑маршрутизация (GeoDNS).

3. Платформенный слой

3.1. Оркестрация контейнеров

Kubernetes (версия 1.28+):

Namespace для каждого сервиса;

Deployment/StatefulSet для сервисов и БД;

Horizontal Pod Autoscaler (HPA) для масштабирования.

3.2. Хранилища данных

PostgreSQL (кластер через Patroni):

Мастер в ЦОД‑1, реплика в ЦОД‑2;

PgBouncer для пула соединений.

Kafka (3 брокер‑ноды):

Репликация между ЦОД;

Zookeeper для координации.

Redis (кластер):

Для кэширования и очередей.

S3‑хранилище (MinIO/AWS S3):

Бэкапы БД;

Файлы пользователей (изображения товаров, документы).

3.3. Сеть и балансировка

Ingress Controller (Nginx/Traefik):

TLS‑терминация;

Rate limiting;

WAF‑фильтрация.

Service Mesh (Istio/Linkerd):

Mutual TLS;

Circuit Breaker;

Tracing (Jaeger).

4. Слой сервисов

Каждый микросервис развёрнут как Deployment в Kubernetes с:

Resource Requests/Limits (CPU/RAM);

Liveness/Readiness Probes;

ConfigMaps/Secrets для конфигурации.

Примеры:

Платежи: 3 пода (HPA при нагрузке > 70%).

Заказы: 5 подов (пиковые нагрузки).

Нотификации: 2 пода + очередь в Redis.

Товары: 4 пода + кэш Redis.

АРМ администратора: 2 пода (UI + API).

5. Интеграционные компоненты

5.1. Брокер сообщений

Apache Kafka (3 брокер‑ноды, 3 реплики на partition):

Топики: orders, payments, notifications, reviews.

Schema Registry для валидации сообщений.

5.2. API Gateway

Kong/Apigee:

Маршрутизация запросов;

Аутентификация (JWT);

Rate limiting.

5.3. Service Discovery

Kubernetes DNS + Consul (для внешних сервисов).

6. Мониторинг и observability

6.1. Технический мониторинг

Prometheus: сбор метрик с:

Kubernetes (kube‑state‑metrics);

PostgreSQL (postgres‑exporter);

Kafka (kafka‑exporter);

Redis (redis‑exporter).

Grafana: дашборды для:

Health сервисов;

Задержки API;

Нагрузки БД;

Kafka lag.

6.2. Логирование

ELK Stack (Elasticsearch + Logstash + Kibana):

Сбор логов всех сервисов;

Алерты на ошибки (5xx, timeouts).

6.3. Распределённый трейсинг

Jaeger: отслеживание запросов через сервисы (trace ID).

6.4. Бизнес‑метрики

FineBI (или Tableau):

Подключение к ClickHouse/PostgreSQL;

Дашборды: конверсия, средний чек, активность пользователей.

7. Безопасность

7.1. Сетевая безопасность

Network Policies в Kubernetes (изоляция namespace).

Firewall между ЦОД.

WAF (Cloudflare/ModSecurity).

7.2. Аутентификация/Авторизация

Keycloak/Auth0 для:

Пользователей (OAuth2);

Админов (RBAC).

JWT‑токены для API.

7.3. Шифрование

TLS 1.3 для всего трафика (Let’s Encrypt).

Шифрование БД (PGP/AES).

Secrets в Kubernetes через Vault.

7.4. Аудит

Логирование действий админов.

Мониторинг аномалий (Falco).

8. Резервное копирование и восстановление
8.1. Бэкапы

БД: ежедневные снапшоты + WAL‑архивация (PgBackRest).

Kafka: резервное копирование топиков (MirrorMaker).

Файлы: репликация в S3 (версиярование).

8.2. Disaster Recovery

RPO (Recovery Point Objective): < 15 мин.

RTO (Recovery Time Objective): < 30 мин.

Автоматическое переключение на ЦОД‑2 при аварии.