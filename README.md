# Spring Cloud Session 4 Inter Microservice Communication ASynchronous using RabbitMQ
In  this tutorial we are going to learn how microservices communicate with each other in asynchronous fashion. In asynchronous 
communication calling microservice will **not wait** till the called microservice responds. This pattern can be achieved 
with message bus infrastructures like Kafka or RabbitMQ.Here we use **Spring Cloud Stream** framework to communicate 
with message bus.

**Overview**
- When report-api microservice receives a request to get employee details it is going to fetch details and write results
to message bus.
- Mail microservice listens on the bus for employee details and when those details are available on bus. It is going to read
send SMS and email.  
- report-api play a role of **Producer**
- mail microservice plays a role of **Consumer**
- RabbitMQ play a role of mediator or **message bus**

**Flow**
- Run registry service on 8761. 
- Run employee-api service on dynamic port. Where it takes employee id and returns employee name.
- Run payroll-api service on dynamic port. Where it takes employee id and returns employee salary.
- Run report-api service on dynamic port. Where it takes employee id and returns employee name and salary by 
directly communicating with employee-api and payroll-api. **It also publishes employee details to message bus**
- Run mail service (it is not a rest api, it a java process and doesnt bind to any port). **Where it subscribes to message bus*** 
for employee details message and sends mail,sms.
- Run Gateway service on 8080 and reverse proxy requests to all the services (employee-api,payroll-api,report-api)
- All the microservices (employee-api,payroll-api,report-api,gateway) when they startup they register their service endpoint (rest api url)
 with registry
- Gateway Spring Cloud load balancer (Client side load balancing) component in Spring Cloud Gateway acts as reverse proxy.
It reads a registry for microservice endpoints and configures routes. 

Important Notes
- Netflix Eureka Server plays a role of Registry. Registry is a spring boot application with Eureka Server as dependency.
- Netflix Eureka Client is present in all the micro services (employee-api,payroll-api,report-api-direct,report-api-via-gateway,gateway) and they discover Eureka
server and register their availability with server.
- Generally Netflix Ribbon Component is used as Client Side load balancer, but it is deprecated project. We will be using
Spring Cloud Load balaner in gateway 
# RabbitMQ Terminology
- **Producer, publisher** A Producer is the application that is sending the messages to the message queue.
- **Consumer** A Consumer is the application that receives the messages from the message queue.
- **Message queue**- A message queue is a queue of messages sent between applications. 
It allow applications to communicate by sending messages to each other.
- **Exchange** An exchange is responsible for the routing of the messages to the different queues. An exchange accepts 
messages from the producer application and routes them to message queues with help of header attributes, bindings, 
and routing keys
- **Ack** When RabbitMQ delivers a message to a consumer, it needs to know when to consider the message successfully 
sent. An ack will acknowledge one or more messages, which tells RabbitMQ that a message/messages has been handled
- **Binding**  A binding is a "link" that you set up to bind a queue to an exchange.
- **Channel** A channel is a virtual connection inside a connection. When you are publishing or consuming messages 
from a queue - it's all done over a channel.
- **Connection**-A connection is a TCP connection between your application and the RabbitMQ broker
# Spring Cloud Stream Concepts
Spring cloud stream abstracts underneath communication  with Messagebus. This helps to foucs on business logic instead of 
 nettigritty of message bus. We can easily switch from RabbitMQ to Kafka etc without code changes.
 - **Bindings** — a collection of interfaces that identify the input and output channels.
- **Channel** — represents the communication pipe between messaging-middleware and the application.
- **StreamListeners**- Listens to messages on Input channel and serializes them to java objects.
 

# Source Code 
``` git clone https://github.com/balajich/spring-cloud-session-3-inter-microservice-communication-sync.git``` 
# Video
[![Spring Cloud Session 2 Microservices Dynamic ports](https://img.youtube.com/vi/5WuallBaMnw/0.jpg)](https://www.youtube.com/watch?v=5WuallBaMnw)
- https://youtu.be/5WuallBaMnw
# Architecture
![architecture](architecture.png "architecture")
# Prerequisite
- JDK 1.8 or above
- Apache Maven 3.6.3 or above
- Vagrant, Virtualbox (To run RabbitMQ Server)
# Start RabbitMQ Server and Build Microservices
We will be running RabbitMQ server inside a docker container. I am running docker container on CentOS7 virtual machine. 
I will be using vagrant to stop or start a virtual machine.
- RabbitMQ Server
    - ``` cd spring-cloud-session-4-inter-microservice-communication-async ```
    - Bring virtual machine up ``` vagrant up ```
    - ssh to virtual machine ```vagrant ssh ```
    - Change folder where docker-compose files is available ```cd /vagrant```
    - Start RabbitMQ Server using docker-compose ``` docker-compose up -d ```
- Java
    - ``` mvn clean install ```
 
# Running components
- Registry: ``` java -jar .\registry\target\registry-0.0.1-SNAPSHOT.jar ```
- Employee API: ``` java -jar .\employee-api\target\employee-api-0.0.1-SNAPSHOT.jar ```
- Payroll API: ``` java -jar .\payroll-api\target\payroll-api-0.0.1-SNAPSHOT.jar ```
- Report API: ``` java -jar .\report-api\target\report-api-0.0.1-SNAPSHOT.jar ```
- Mail App: ``` java -jar .\mail-client\target\mail-client-0.0.1-SNAPSHOT.jar ```
- Gateway: ``` java -jar .\gateway\target\gateway-0.0.1-SNAPSHOT.jar ``` 

# Using curl to test environment
**Note I am running CURL on windows, if you have any issue. Please use postman client, its collection is available 
at spring-cloud-session-3-inter-microservice-communication-sync.postman_collection.json**
- Get employee report using report api ( direct): ``` curl -s -L  http://localhost:8080/report-api-direct/100 ```
- Get employee report using report-api-via-gateway: ``` curl -s -L  http://localhost:8080/report-api-via-gateway/100 ```

**Note: In real world we favour to call microservices via a gateway even for inter communication. So I recommend using 
the  microservice report-api-via-gateway**  
# Code
In this section will focus only on report-api and how communicates employee-api,payroll-api. 

*ReportController* in app **report-api-direct**. RestTemplate calls eureka ribbon client which fetches "employee-api,payroll-api" information from registry 
and calls the microservices directly. The **@LoadBalanced** annotation makes ribbon client to round-robbin requests if there are multiple instances of them.
```java
@Autowired
    RestTemplate restTemplate;

    @RequestMapping(value = "/report-api-direct/{employeeId}", method = RequestMethod.GET)
    public Employee getEmployeeDetails(@PathVariable int employeeId) {
        logger.info(String.format("Getting Complete Details of Employee with id %s", employeeId));
        //Get employee name from employee-api
        Employee responseEmployeeNameDetails = restTemplate.getForEntity("http://employee-api/employee/" + employeeId, Employee.class).getBody();
        //Get employee salary from payroll-api
        Employee responseEmployeePayDetails = restTemplate.getForEntity("http://payroll-api/payroll/" + employeeId, Employee.class).getBody();
        return new Employee(responseEmployeeNameDetails.getId(), responseEmployeeNameDetails.getName(), responseEmployeePayDetails.getSalary());
    }

    @Bean
    @LoadBalanced
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }
```
*ReportController* in app **report-api-via-gateway**. RestTemplate calls eureka ribbon client which fetches "gateway" information from registry 
and calls the employee,payroll api via gateway. The **@LoadBalanced** annotation makes ribbon client to round-robbin requests if there are multiple instances of them.
```java
  @Autowired
    RestTemplate restTemplate;

    @RequestMapping(value = "/report-api-via-gateway/{employeeId}", method = RequestMethod.GET)
    public Employee getEmployeeDetails(@PathVariable int employeeId) {
        logger.info(String.format("Getting Complete Details of Employee with id %s", employeeId));
        //Get employee name from employee-api via gateway
        Employee responseEmployeeNameDetails = restTemplate.getForEntity("http://gateway/employee/" + employeeId, Employee.class).getBody();
        //Get employee salary from payroll-api via gateway
        Employee responseEmployeePayDetails = restTemplate.getForEntity("http://gateway/payroll/" + employeeId, Employee.class).getBody();
        return new Employee(responseEmployeeNameDetails.getId(), responseEmployeeNameDetails.getName(), responseEmployeePayDetails.getSalary());
    }

    @Bean
    @LoadBalanced
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }
```
**Gateway**
```yaml
cloud:
    loadbalancer:
      ribbon:
        enabled: false
    gateway:
      routes:
        - id: employee-api
          uri: lb://EMPLOYEE-API
          predicates:
            - Path=/employee/**
        - id: payroll-api
          uri: lb://PAYROLL-API
          predicates:
            - Path=/payroll/**
        - id: report-api-direct
          uri: lb://REPORT-API-DIRECT
          predicates:
            - Path=/report-api-direct/**
        - id: report-api-via-gateway
          uri: lb://REPORT-API-VIA-GATEWAY
          predicates:
            - Path=/report-api-via-gateway/**
```

# Next Steps
- Inter microservice communication in asynchronous fashion
# References
- https://www.baeldung.com/spring-cloud-stream
- Spring Microservices in Action by John Carnell 
- Hands-On Microservices with Spring Boot and Spring Cloud: Build and deploy Java microservices 
using Spring Cloud, Istio, and Kubernetes -Magnus Larsson
- https://www.cloudamqp.com/blog/2017-07-25-RabbitMQ-and-AMQP-concepts-glossary.html 
# Next Tutorial
https://github.com/balajich/spring-cloud-session-4-inter-microservice-communication-async