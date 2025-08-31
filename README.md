# Discovery Service (Eureka Server)

## 📋 Описание

Discovery Service - это сервер обнаружения сервисов, построенный на Netflix Eureka. Он обеспечивает централизованный реестр всех микросервисов в системе и автоматическое обнаружение сервисов.

## 🎯 Основные функции

- Регистрация и обнаружение микросервисов
- Health checking зарегистрированных сервисов
- Load balancing между экземплярами сервисов
- Fault tolerance и self-healing
- Web dashboard для мониторинга сервисов

## ⚙️ Технический стек

- **Java 17**
- **Spring Boot 3.5.3**
- **Spring Cloud Netflix Eureka Server 4.3.0**
- **Spring Cloud Config Client 4.3.0**
- **Maven**

## 🚀 Запуск сервиса

### Предварительные условия
- Config Server должен быть запущен (http://localhost:8888)
- Java 17+
- Maven 3.6+

### Локальный запуск

```bash
# Клонирование репозитория
git clone <repository-url>
cd services/discovery

# Сборка проекта
./mvnw clean install

# Запуск сервиса
./mvnw spring-boot:run
```

### Docker запуск

```bash
docker build -t discovery-service .
docker run -p 8761:8761 discovery-service
```

## 🔧 Конфигурация

### application.yml
```yaml
spring:
  config:
    import: optional:configserver:http://localhost:8888
  application:
    name: discovery-service
eureka:
  instance:
    hostname: localhost
  client:
    register-with-eureka: false
    fetch-registry: false
    serviceUrl:
      defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka/
server:
  port: 8761
```

### Конфигурация в Config Server (discovery-service.yml)
```yaml
eureka:
  instance:
    hostname: localhost
  client:
    register-with-eureka: false
    fetch-registry: false
    serviceUrl:
      defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka/
server:
  port: 8761
```

## 🌐 Интерфейсы

### Eureka Dashboard
Веб-интерфейс для мониторинга зарегистрированных сервисов:
```
http://localhost:8761
```

Отображает:
- Список всех зарегистрированных сервисов
- Статус каждого сервиса (UP/DOWN)
- Количество экземпляров каждого сервиса
- Время последнего heartbeat
- Метаданные сервисов

### REST API Endpoints

#### Информация о сервисах
```http
GET http://localhost:8761/eureka/apps
Accept: application/json
```

#### Информация о конкретном сервисе
```http
GET http://localhost:8761/eureka/apps/{SERVICE_NAME}
Accept: application/json
```

#### Регистрация сервиса
```http
POST http://localhost:8761/eureka/apps/{SERVICE_NAME}
Content-Type: application/json
```

#### Health check
```http
GET http://localhost:8761/actuator/health
```

## 📊 Мониторинг и метрики

### Ключевые метрики
- **Registered instances** - количество зарегистрированных экземпляров
- **Available instances** - количество доступных экземпляров
- **Unavailable instances** - количество недоступных экземпляров
- **Heartbeats received** - количество полученных heartbeat'ов
- **Renewals threshold** - порог обновлений

### Логирование
Для включения подробного логирования:
```yaml
logging:
  level:
    com.netflix.eureka: DEBUG
    com.netflix.discovery: DEBUG
```

### Health Indicators
Eureka предоставляет следующие health indicators:
- **eureka** - статус Eureka сервера
- **diskSpace** - состояние дискового пространства
- **refreshScope** - статус refresh scope

## 🔧 Настройки производительности

### Оптимизация времени обнаружения сервисов

```yaml
eureka:
  server:
    # Отключить самозащиту в development окружении
    enable-self-preservation: false
    # Интервал очистки неактивных сервисов
    eviction-interval-timer-in-ms: 4000
  instance:
    # Интервал отправки heartbeat
    lease-renewal-interval-in-seconds: 10
    # Время ожидания heartbeat
    lease-expiration-duration-in-seconds: 30
```

### Настройки для production

```yaml
eureka:
  server:
    enable-self-preservation: true
    expected-number-of-clients-sending-renews: 5
    renewal-percent-threshold: 0.85
  instance:
    lease-renewal-interval-in-seconds: 30
    lease-expiration-duration-in-seconds: 90
```

## 🔗 Интеграция с клиентскими сервисами

### Регистрация клиентского сервиса

Клиентские сервисы должны включать в `pom.xml`:
```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```

И в `application.yml`:
```yaml
eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka
  instance:
    prefer-ip-address: true
    lease-renewal-interval-in-seconds: 10
    lease-expiration-duration-in-seconds: 30
```

### Использование @LoadBalanced RestTemplate

```java
@Bean
@LoadBalanced
public RestTemplate restTemplate() {
    return new RestTemplate();
}

// Использование
restTemplate.getForObject("http://CUSTOMER-SERVICE/api/v1/customers", String.class);
```

### Использование OpenFeign

```java
@FeignClient(name = "customer-service")
public interface CustomerClient {
    @GetMapping("/api/v1/customers/{id}")
    CustomerResponse getCustomer(@PathVariable("id") String id);
}
```

## 🐛 Устранение неполадок

### Частые проблемы

1. **Сервисы не регистрируются в Eureka**
   ```
   Решение:
   - Убедитесь, что Discovery Service запущен
   - Проверьте URL в eureka.client.service-url.defaultZone
   - Проверьте зависимости spring-cloud-starter-netflix-eureka-client
   ```

2. **Eureka Dashboard показывает красное предупреждение**
   ```
   EMERGENCY! EUREKA MAY BE INCORRECTLY CLAIMING INSTANCES ARE UP WHEN THEY'RE NOT.
   
   Решение:
   - В development окружении: отключите self-preservation
   - В production: проверьте настройки renewal-percent-threshold
   ```

3. **Медленное обнаружение новых сервисов**
   ```
   Решение:
   - Уменьшите lease-renewal-interval-in-seconds
   - Уменьшите eviction-interval-timer-in-ms
   - Настройте ribbon.ServerListRefreshInterval
   ```

4. **Сервисы не удаляются из реестра после остановки**
   ```
   Решение:
   - Проверьте lease-expiration-duration-in-seconds
   - Убедитесь в корректном graceful shutdown
   - Проверьте eviction-interval-timer-in-ms
   ```

## 🔐 Безопасность

### Базовая аутентификация

```yaml
spring:
  security:
    user:
      name: eureka-user
      password: eureka-password

eureka:
  client:
    service-url:
      defaultZone: http://eureka-user:eureka-password@localhost:8761/eureka
```

```

## 🏗️ Высокая доступность

### Настройка кластера Eureka

#### Eureka Server 1
```yaml
eureka:
  instance:
    hostname: eureka-server-1
  client:
    service-url:
      defaultZone: http://eureka-server-2:8762/eureka/
server:
  port: 8761
```

#### Eureka Server 2
```yaml
eureka:
  instance:
    hostname: eureka-server-2
  client:
    service-url:
      defaultZone: http://eureka-server-1:8761/eureka/
server:
  port: 8762
```

#### Клиентская конфигурация для кластера
```yaml
eureka:
  client:
    service-url:
      defaultZone: http://eureka-server-1:8761/eureka/,http://eureka-server-2:8762/eureka/
```

## 📝 Структура проекта

```
discovery/
├── src/
│   └── main/
│       ├── java/
│       │   └── kg/manurov/discovery/
│       │       └── DiscoveryApplication.java
│       └── resources/
│           └── application.yml
├── target/
├── pom.xml
└── README.md
```