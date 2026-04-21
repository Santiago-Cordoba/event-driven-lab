# Event-Driven-Lab
Laboratorio práctico en Java para aplicar la arquitectura **Event-Driven** mediante la comunicación asíncrona entre microservicios utilizando **Spring Boot, RabbitMQ, Docker Compose y GitHub Codespaces**.

---

# Objetivo

Implementar dos microservicios Spring Boot que se comuniquen de forma asíncrona a través de **RabbitMQ**, orquestados con **Docker Compose** y desplegados en un entorno online gratuito.

Este laboratorio demuestra cómo estructurar un proyecto de arquitectura orientada a eventos con:

* Microservicio **Productor** con API REST para publicar mensajes
* Microservicio **Consumidor** que escucha y procesa mensajes de la cola
* Comunicación asíncrona mediante **RabbitMQ**
* Configuración de **Exchange, Queue y Binding**
* Orquestación con **Docker Compose**
* Despliegue de imágenes en **Docker Hub**

---

# Tecnologías Utilizadas

* Java 17+
* Spring Boot
* Spring AMQP (RabbitMQ)
* RabbitMQ + Management UI
* Docker & Docker Compose
* Maven
* GitHub Codespaces / Killercoda

---

# Metodología Aplicada: Event-Driven Architecture

El desarrollo sigue el flujo clásico de una **arquitectura orientada a eventos**:

1. El **Productor** expone un endpoint REST y publica mensajes en un **Exchange**
2. RabbitMQ enruta el mensaje a la **Queue** correspondiente mediante una **Routing Key**
3. El **Consumidor** escucha la cola y procesa cada mensaje recibido
4. Todo el sistema se orquesta mediante **Docker Compose** sobre una red compartida

---

# Estructura del Repositorio

```
event-driven-lab/
│
├── producer-service/
│   ├── src/
│   │   └── main/java/com/eci/arcn/producer_service/
│   │       ├── config/
│   │       │   └── RabbitMQConfig.java
│   │       └── controller/
│   │           └── MessageController.java
│   ├── src/main/resources/
│   │   └── application.properties
│   ├── pom.xml
│   └── Dockerfile
│
├── consumer-service/
│   ├── src/
│   │   └── main/java/com/eci/arcn/consumer_service/
│   │       ├── config/
│   │       │   └── RabbitMQConfig.java
│   │       └── listener/
│   │           └── MessageListener.java
│   ├── src/main/resources/
│   │   └── application.properties
│   ├── pom.xml
│   └── Dockerfile
│
├── .devcontainer/
│   └── devcontainer.json
├── docker-compose.yml
└── README.md
```

---

# Servicios Implementados

## 1️⃣ Producer Service

Microservicio que expone un endpoint REST para publicar mensajes en RabbitMQ.

```java
@PostMapping("/send")
public String sendMessage(@RequestParam String message) {
    rabbitTemplate.convertAndSend(exchangeName, routingKey, message);
    return "Mensaje '" + message + "' enviado!";
}
```

**Endpoint disponible:**

```
POST http://localhost:8080/api/messages/send?message=Hola
```

---

## 2️⃣ Consumer Service

Microservicio que escucha la cola de RabbitMQ y procesa cada mensaje recibido.

```java
@RabbitListener(queues = "${app.rabbitmq.queue}")
public void receiveMessage(String message) {
    log.info("Mensaje recibido: '{}'", message);
    System.out.println(">>> Mensaje Procesado: " + message);
}
```

---

## 3️⃣ Configuración RabbitMQ

Tanto el productor como el consumidor declaran la cola. El productor además configura el **Exchange** y el **Binding**:

```java
@Bean
Queue queue() {
    return new Queue(queueName, true);
}

@Bean
DirectExchange exchange() {
    return new DirectExchange(exchangeName);
}

@Bean
Binding binding(Queue queue, DirectExchange exchange) {
    return BindingBuilder.bind(queue).to(exchange).with(routingKey);
}
```

---

# Configuración de application.properties

Ambos servicios comparten la misma configuración de conexión a RabbitMQ. El host `rabbitmq` corresponde al nombre del servicio en Docker Compose:

```properties
spring.rabbitmq.host=rabbitmq
spring.rabbitmq.port=5672
spring.rabbitmq.username=guest
spring.rabbitmq.password=guest

app.rabbitmq.exchange=messages.exchange
app.rabbitmq.queue=messages.queue
app.rabbitmq.routingkey=messages.routingkey
```

---

# Docker Compose

Los tres servicios se orquestan en una red compartida `event_network`:

```yaml
version: '3.8'

services:
  rabbitmq:
    image: rabbitmq:management
    ports:
      - "5672:5672"
      - "15672:15672"
    networks:
      - event_network

  producer:
    image: <tudockerhubusername>/producer-service
    ports:
      - "8080:8080"
    depends_on:
      - rabbitmq
    environment:
      - SPRING_RABBITMQ_HOST=rabbitmq
    networks:
      - event_network

  consumer:
    image: <tudockerhubusername>/consumer-service
    depends_on:
      - rabbitmq
    environment:
      - SPRING_RABBITMQ_HOST=rabbitmq
    networks:
      - event_network

networks:
  event_network:
    driver: bridge
```

---

# Publicar Imágenes en Docker Hub

Ejecuta los siguientes pasos para cada servicio:

```bash
# 1. Compilar el proyecto
mvn package -DskipTests

# 2. Construir la imagen
docker build -t producer-service .

# 3. Etiquetar la imagen
docker tag producer-service TU_USUARIO/producer-service

# 4. Iniciar sesión en Docker Hub
docker login -u TU_USUARIO

# 5. Subir la imagen
docker push TU_USUARIO/producer-service
```

> 🔑 Genera tu **Personal Access Token** en Docker Hub: **Account Settings → Security → New Access Token**

---

# Cómo Ejecutar el Laboratorio

### Prerrequisitos

* Cuenta en GitHub (para Codespaces)
* Cuenta en Docker Hub
* Java 17+ y Maven (disponibles en el devcontainer)

### Levantar todos los servicios

```bash
docker-compose up -d
```

### Verificar que los contenedores están corriendo

```bash
docker-compose ps
```

### Detener los servicios

```bash
docker-compose down
```

---

# Probar el Flujo de Eventos

### Enviar un mensaje desde el Productor

```bash
curl -X POST "http://localhost:8080/api/messages/send?message=HolaMundo"
```

Respuesta esperada:

```
Mensaje 'HolaMundo' enviado!
```

### Verificar que el Consumidor recibió el mensaje

```bash
docker-compose logs consumer
```

Salida esperada:

```
Mensaje recibido: 'HolaMundo'
>>> Mensaje Procesado: HolaMundo
```

---

# RabbitMQ Management UI

Accede a la interfaz de administración de RabbitMQ en el puerto **15672**:

```
http://localhost:15672
```

**Credenciales:** `guest` / `guest`

Desde la pestaña **Queues** puedes monitorear el estado de `messages.queue` y el flujo de mensajes en tiempo real.

---

# Prueba de su Funcionamiento

<img width="1268" height="380" alt="docker1" src="https://github.com/user-attachments/assets/a5c826bf-b700-4532-9e0b-b5866737b76d" />

<img width="1091" height="36" alt="docker2" src="https://github.com/user-attachments/assets/8e68ecea-6a7a-4028-97cd-7f1c134f1e42" />

<img width="1559" height="445" alt="docker3" src="https://github.com/user-attachments/assets/9cb628b7-dfd0-46ea-88ae-41f866259f5c" />


---

# Conclusión

Este laboratorio demuestra la implementación de una **arquitectura orientada a eventos** con Spring Boot y RabbitMQ, utilizando Docker Compose para orquestar todos los servicios en una red compartida.

El enfoque **Event-Driven** desacopla los microservicios, permitiendo que el productor y el consumidor evolucionen de forma independiente, mejorando la escalabilidad, resiliencia y mantenibilidad del sistema.
