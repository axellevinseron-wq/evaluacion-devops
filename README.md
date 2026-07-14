# 🛒 EcoMarket - Arquitectura de Microservicios

![Java](https://img.shields.io/badge/Java-17-orange?logo=openjdk&logoColor=white)
![Spring Boot](https://img.shields.io/badge/Spring%20Boot-4.0.6-brightgreen?logo=springboot)
![Docker](https://img.shields.io/badge/Docker-Compose-2496ED?logo=docker&logoColor=white)
![Prometheus](https://img.shields.io/badge/Prometheus-monitoring-E6522C?logo=prometheus&logoColor=white)
![Grafana](https://img.shields.io/badge/Grafana-dashboards-F46800?logo=grafana&logoColor=white)
![Loki](https://img.shields.io/badge/Loki-logs-2E3137?logo=grafana&logoColor=white)

Sistema e-commerce con 10 microservicios independientes, MySQL dedicada por servicio y Docker Compose. Incluye un stack de observabilidad (**Prometheus + Grafana + Loki**) ya configurado y probado sobre **ms-carrito**.

## 📋 Stack Técnico

| Componente | Versión |
|-----------|---------|
| Java | 17 |
| Spring Boot | 4.0.6 |
| Spring Cloud | 2025.1.1 |
| MySQL | 8.0 |
| Lombok | 1.18.30 |
| Flyway | 9.22.3 |
| OpenFeign | Cloud |
| Docker | 3.8+ |
| Prometheus | latest |
| Grafana | latest |
| Loki / Promtail | 2.9.8 |

## 🏗️ Arquitectura de Microservicios

| MS | Puerto | BD | Responsabilidad |
|----|--------|----|----|
| **ms-catalogo** | 8081 | ms_catalogo | Productos, categorías |
| **ms-inventario** | 8084 | ms_inventario | Stock, reservas |
| **ms-carrito** | 8083 | ms_carrito | Carrito usuario, Feign→catalogo |
| **ms-pedidos** | 8082 | ms_pedidos | Órdenes, estados (PAGADA/CANCELADA) |
| **ms-promociones** | 8085 | ms_promociones | Descuentos, vigencia |
| **ms-pagos** | 8086 | ms_pagos | Transacciones |
| **ms-usuarios** | 8087 | ms_usuarios | Autenticación |
| **ms-envios** | 8088 | ms_envios | Tracking |
| **ms-notificaciones** | 8089 | ms_notificaciones | Email, SMS |
| **ms-clientes** | 8090 | ms_clientes | Perfiles cliente |

## 📊 Observabilidad: levantar solo `ms-carrito` (Prometheus + Grafana + Loki)

`ms-carrito` es el microservicio instrumentado para esta evaluación con un stack de observabilidad completo:

- **Spring Boot Actuator + Micrometer** → expone métricas en `/actuator/prometheus`
- **Prometheus** → scrapea esas métricas cada 15s (`prometheus.yml`)
- **Loki + Promtail** → centralizan los logs del contenedor automáticamente (vía socket de Docker)
- **Grafana** → ya viene con los datasources de Prometheus y Loki preconfigurados (`grafana/provisioning/datasources`)

No es necesario levantar los otros 9 microservicios para probar este stack. Con `mysql` (BD de `ms-carrito`) y los servicios de observabilidad es suficiente.

### 1. Requisitos previos

- **Docker** & **Docker Compose** v2+
- **Java 17+** y **Maven 3.8+** (opcional, solo si quieres compilar fuera de Docker)

### 2. Levantar únicamente `ms-carrito` + observabilidad

```bash
# Desde la raíz del proyecto
cd FullStack_evaluacion

# Levanta mysql, ms-carrito, prometheus, loki, promtail y grafana
# (el resto de microservicios NO se levantan)
docker-compose up -d --build mysql ms-carrito prometheus loki promtail grafana

# Verificar que todo esté arriba y "healthy"
docker-compose ps
```

> La primera vez usa `--build` para compilar la imagen de `ms-carrito`. En arranques posteriores puedes omitirlo: `docker-compose up -d mysql ms-carrito prometheus loki promtail grafana`.

### 3. Verificar que ms-carrito esté exponiendo métricas

```bash
# Health check
curl http://localhost:8083/actuator/health

# Métricas en formato Prometheus
curl http://localhost:8083/actuator/prometheus

# Generar tráfico de prueba (requiere ms-catalogo activo para el Feign;
# si no está, el circuit breaker responde con el fallback controlado)
curl -X POST http://localhost:8083/api/carrito \
  -H "Content-Type: application/json" \
  -d '{"clienteId":1,"productoId":1,"cantidad":2}'
```

### 4. Acceder a cada herramienta

| Herramienta | URL | Credenciales |
|---|---|---|
| **ms-carrito** (API) | http://localhost:8083 | - |
| **ms-carrito** métricas | http://localhost:8083/actuator/prometheus | - |
| **Prometheus** (UI / Targets) | http://localhost:9090/targets | - |
| **Grafana** | http://localhost:3000 | `admin` / `admin` |
| **Loki** (API interna, vía Grafana) | http://localhost:3100 | - |

En **Prometheus** → `Status > Targets` deberías ver el job `ms-carrito` en estado `UP`.

En **Grafana**:
1. Entra con `admin` / `admin`.
2. Los datasources **Prometheus** y **Loki** ya están provisionados (no hay que configurarlos a mano).
3. Ve a **Explore** → elige el datasource **Prometheus** y consulta, por ejemplo: `http_server_requests_seconds_count{job="ms-carrito"}`.
4. Cambia el datasource a **Loki** y consulta los logs del contenedor con: `{service="ms-carrito"}`.

### 5. Detener solo este stack

```bash
docker-compose stop mysql ms-carrito prometheus loki promtail grafana

# o eliminar contenedores y red (mantiene el volumen de mysql)
docker-compose down
```

## 📦 Requisitos Previos (instalación completa de los 10 microservicios)

- **Docker** & **Docker Compose** v3.8+
- **Maven** 3.8+ (compilar JARs)
- **Java 17+**
- **Git**

## 🚀 Instalación y Ejecución

### 1. Clonar y compilar todos los servicios
```bash
cd PROYECTO
mvn clean package -DskipTests
```

### 2. Verificar JARs compilados
```bash
ls -la ms-catalogo/target/ms-catalogo-1.0.0.jar
ls -la ms-pedidos/target/ms-pedidos-1.0.0.jar
ls -la ms_carrito/target/ms_carrito-1.0.0.jar
```

### 3. Levantar infraestructura con Docker
```bash
docker-compose up -d
docker-compose ps  # Verificar servicios
```

### 4. Verificar conectividad
```bash
# MySQL (localhost:3306, user: root, pass: root)
mysql -h localhost -u root -proot -e "SHOW DATABASES;"

# PhpMyAdmin (http://localhost:8080)

# API Catalogo
curl http://localhost:8081/api/catalogo

# API Carrito (con Feign a catalogo)
curl -X POST http://localhost:8083/api/carrito \
  -H "Content-Type: application/json" \
  -d '{"clienteId":1,"productoId":1,"cantidad":2}'

# API Pedidos
curl http://localhost:8082/api/pedidos
```

## 🔗 Endpoints Críticos

### ms-catalogo (8081) - Catalog Service
```
GET    /api/catalogo                  → Listar categorías con productos
POST   /api/catalogo                  → Crear categoría [ADMIN]
GET    /api/catalogo/productos/{id}   → Obtener producto por ID [Consumidor: ms-carrito Feign]
```

### ms-carrito (8083) - Shopping Cart Service
```
POST   /api/carrito                   → Crear item (valida precio vía Feign)
GET    /api/carrito                   → Listar items carrito
GET    /api/carrito/{id}              → Obtener item específico
DELETE /api/carrito/{id}              → Eliminar item
```

**Flujo:** `POST /api/carrito` → ProductoFeignClient → ms-catalogo (8081) → validar precio/existencia

### ms-pedidos (8082) - Orders Service
```
POST   /api/pedidos                   → Crear pedido (estado: PENDIENTE)
GET    /api/pedidos                   → Listar pedidos
GET    /api/pedidos/{id}              → Obtener pedido
PUT    /api/pedidos/{id}/estado       → Cambiar estado [PAGADA/CANCELADA] + LOGS
```

**Estados válidos:**
- `PENDIENTE` → `PAGADA` o `CANCELADA`
- `PAGADA` → `ENVIADO` o `CANCELADA`
- `ENVIADO` → `ENTREGADO`
- `ENTREGADO` → (sin cambios)
- `CANCELADA` → (sin cambios)

### ms-inventario (8084) - Inventory Service
```
POST   /api/inventario/reservar       → Validar y reservar stock
GET    /api/inventario/producto/{id}  → Consultar disponibilidad
PUT    /api/inventario/liberar        → Devolver stock (si cancela pedido)
```

## 🏛️ Patrones Implementados

### 1. Resiliencia Inter-Servicio (Circuit Breaker)
**Ubicación:** ms-carrito → ProductoFeignClient

- **Timeout:** 5s (connect + read)
- **Fallback:** ProductoFeignClientFallback (devuelve error controlado)
- **Circuit Breaker:** Resilience4j (abre si 50% de llamadas fallan)

```java
@FeignClient(name = "ms-catalogo", 
    url = "http://localhost:8080",
    fallback = ProductoFeignClientFallback.class)
```

**Cómo probar fallo:**
```bash
docker-compose stop ms-catalogo
curl -X POST http://localhost:8083/api/carrito \
  -d '{"clienteId":1,"productoId":999,"cantidad":1}' 
# Respuesta: 400 + "Catálogo temporalmente no disponible"
```

### 2. Trazabilidad (@Slf4j)
**Ubicación:** ms-pedidos Service

**Logs generados:**
- `log.info()` - Creación pedido, cambio estado (PAGADA/CANCELADA)
- `log.warn()` - Transiciones inválidas, stock bajo
- `log.error()` - Fallos Feign, BD caída

**Ejemplo salida:**
```
INFO - Guardando nuevo pedido para cliente: 1
INFO - ✓ PEDIDO PAGADO: AUDITORIA - Pedido 1: Pendiente -> Pagado
ERROR - Excepción de negocio - Código: TRANSICION_INVALIDA, Mensaje: Transición inválida de ENTREGADO a PAGADA
```

### 3. Validación (Bean Validation)
**Global:** GlobalExceptionHandler (@RestControllerAdvice)

```java
@ExceptionHandler(MethodArgumentNotValidException.class)
// → 400 + ErrorResponseDTO { code: "VALIDATION_ERROR", details: {...}, timestamp, status }

@ExceptionHandler(BusinessException.class)
// → 400 + ErrorResponseDTO { code: "PEDIDO_NO_ENCONTRADO", message, status }
```

**DTOs validados:**
- CartItemDTO: @NotNull productId, @Min(1) quantity, @Min(0) price
- ProductoDTO: @NotBlank nombre, @Min(0) precio
- PedidoDTO: @NotNull clienteId

### 4. Persistencia (JPA + Flyway)
**Ubicación:** Cada servicio tiene su BD independiente

- **DDL:** `ddl-auto=validate` (no auto-create en prod)
- **Migraciones:** Flyway (classpath:db/migration/)
- **Script automático:** Al levantar container, Flyway crea tablas

## 🔍 Monitoreo y Logs

### Ver logs en tiempo real
```bash
# Catalogo
docker-compose logs -f ms-catalogo

# Pedidos (auditoría de cambios de estado)
docker-compose logs -f ms-pedidos | grep -i "PAGADO\|CANCELADO"

# Carrito (consumo de Feign)
docker-compose logs -f ms-carrito

# Todos los servicios
docker-compose logs -f --tail=50
```

### Buscar errores
```bash
docker-compose logs | grep -i "error\|exception"
```

### Verificar estado MySQL
```bash
docker-compose logs mysql
docker exec ecomarket_mysql_db mysql -u root -proot -e "SHOW DATABASES;"
```

## 🛠️ Troubleshooting

| Problema | Síntoma | Solución |
|----------|---------|----------|
| "Connection refused" en ms-carrito | Error 500 POST /api/carrito | Esperar 30s a MySQL, ver `docker-compose logs mysql` |
| ms-catalogo no responde | 400 "Catálogo no disponible" | Fallback activado, revisar `docker-compose logs ms-catalogo` |
| Base de datos no creada | Error "Unknown database" | Verificar Flyway scripts en `resources/db/migration/`, ver logs Flyway |
| Puerto ya en uso | "bind: address already in use" | Cambiar puerto en docker-compose.yml: `"8081:8080"` → `"8091:8080"` |
| Feign timeout | 500 después de 5s | Incrementar timeout en application.properties: `feign.client.config.default.readTimeout=10000` |

## 🧪 Testing Flujo Completo

### 1. Crear categoría + producto (ms-catalogo)
```bash
curl -X POST http://localhost:8081/api/catalogo \
  -H "Content-Type: application/json" \
  -d '{
    "nombre": "GPUs",
    "productos": [
      {"nombre": "RTX 4090", "precio": 1599.99}
    ]
  }'
# Response: Categoria { id: 1, productos: [{ id: 1, ... }] }
```

### 2. Agregar a carrito (ms-carrito con Feign)
```bash
curl -X POST http://localhost:8083/api/carrito \
  -H "Content-Type: application/json" \
  -d '{
    "clienteId": 1,
    "productoId": 1,
    "cantidad": 2
  }'
# Response: ItemCarrito { id: 1, productoId: 1, precio: 1599.99 (del Feign) }
```

### 3. Crear pedido (ms-pedidos)
```bash
curl -X POST http://localhost:8082/api/pedidos \
  -H "Content-Type: application/json" \
  -d '{
    "clienteId": 1
  }'
# Response: Pedido { id: 1, estado: PENDIENTE, ... }
```

### 4. Cambiar estado a PAGADA + logs
```bash
curl -X PUT "http://localhost:8082/api/pedidos/1/estado?estado=PAGADA"
# Response: Pedido { id: 1, estado: PAGADA, ... }
# Logs: "✓ PEDIDO PAGADO: AUDITORIA - Pedido 1: Pendiente -> Pagado"
```

## 📐 Forma de Entrega

**Entregables:**
- ✅ Código fuente compilado: `ms-*/target/*.jar`
- ✅ docker-compose.yml ejecutable
- ✅ BD autoinicializada por Flyway
- ✅ Endpoints funcionales inter-servicio (Feign)
- ✅ Logs de auditoría para cambios de estado
- ✅ Manejo de errores global (@RestControllerAdvice)

**Verificación:**
```bash
docker-compose up -d
# Esperar 30s
curl http://localhost:8081/api/catalogo  # OK → 200
curl http://localhost:8082/api/pedidos    # OK → 200
curl http://localhost:8083/api/carrito    # OK → 200
```

## 📚 Documentación Adicional

- **GAP_ANALYSIS.md** - Análisis de brechas vs rúbrica
- **DOCKER_SETUP.md** - Guía detallada Docker Compose
- **pom.xml (padre)** - Dependencias comunes, versiones

---

**Última actualización:** 2026-06-28  
**Mantenedor:** EcoMarket Team

Prueba GitHub Actions CD
Prueba despliegue automático