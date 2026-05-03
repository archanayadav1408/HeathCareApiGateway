# Healthcare API Gateway

Single entry point for all client requests in the Healthcare Appointment System.
Routes incoming requests to the appropriate downstream microservice and provides
cross-cutting concerns like rate limiting and monitoring.

Part of the Healthcare Appointment System built for SEZG583 - Scalable Services assignment.

## Tech Stack
- Java 17
- Spring Boot 3.5.13
- Spring Cloud Gateway
- Spring Boot Actuator
- Maven

## Port
Runs on **port 8080**

## Routing Configuration

| Client Request | Routes To | Service Port |
|----------------|-----------|--------------|
| `/api/patients/**` | Patient Service | 8081 |
| `/api/appointments/**` | Appointment Service | 8082 |

## Architecture
```
Client (Postman / Browser)
          │
          ▼ port 8080
    [API Gateway]
     /           \
    ▼             ▼
Patient       Appointment
Service       Service
(8081)        (8082)
```

## Sample Requests via Gateway

All requests go through port 8080 — no need to know internal service ports.

### Register Patient
```
POST http://localhost:8080/api/patients
```

### Book Appointment
```
POST http://localhost:8080/api/appointments
```

### Get Patient
```
GET http://localhost:8080/api/patients/1
```

### Get Appointment
```
GET http://localhost:8080/api/appointments/1
```

## Actuator Endpoints

| Endpoint | Description |
|----------|-------------|
| `GET /actuator/health` | Health status of gateway |
| `GET /actuator/gateway/routes` | List all configured routes |
| `GET /actuator/metrics` | Application metrics |

## Running Locally

### Prerequisites
- Java 17
- Maven 3.9+
- Patient Service running on port 8081
- Appointment Service running on port 8082

### Run the service
```bash
mvn clean package -DskipTests
java -jar target/api-gateway-0.0.1-SNAPSHOT.jar
```

## Running via Docker
```bash
docker build -t api-gateway:1.0 .
docker run -p 8080:8080 api-gateway:1.0
```

## Project Structure
```
src/main/java/com/healthcare/gateway/
└── ApiGatewayApplication.java

src/main/resources/
└── application.yml    # All routing config lives here — no Java code needed
```

## Group Members
| Name | ID |
|------|----|
| Member 1 | ID |
| Member 2 | ID |
| Member 3 | ID |
| Member 4 | ID |
