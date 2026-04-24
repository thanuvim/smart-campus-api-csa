# Smart Campus Sensor & Room Management API

### 5COSC022W - Client Server Architectures Coursework 2025/26
      Author: Thanuvi Muthukumarana 
      UOW ID: w2120145
      IIT ID: 2024868
---

## Overview

This project is a RESTful API built using JAX-RS (Jersey) deployed as a Web Application
on Apache Tomcat. The API manages Rooms and Sensors across a university Smart Campus.
All data is stored in-memory using ConcurrentHashMap. No database is used.

*Base URL:* http://localhost:8080/smartcampus-api/api/v1/

## Sample curl Commands

**1. Discovery**
```bash
curl -X GET http://localhost:8080/smartcampus-api/api/v1/
```

**2. Create a Room**
```bash
curl -X POST http://localhost:8080/smartcampus-api/api/v1/rooms \
  -H "Content-Type: application/json" \
  -d "{\"id\":\"LIB-301\",\"name\":\"Library Quiet Study\",\"capacity\":80}"
```

**3. Get All Rooms**
```bash
curl -X GET http://localhost:8080/smartcampus-api/api/v1/rooms
```

**4. Get Room by ID**
```bash
curl -X GET http://localhost:8080/smartcampus-api/api/v1/rooms/LIB-301
```

**5. Create a Sensor**
```bash
curl -X POST http://localhost:8080/smartcampus-api/api/v1/sensors \
  -H "Content-Type: application/json" \
  -d "{\"id\":\"TEMP-001\",\"type\":\"Temperature\",\"status\":\"ACTIVE\",\"currentValue\":22.5,\"roomId\":\"LIB-301\"}"
```

**6. Filter Sensors by Type**
```bash
curl -X GET "http://localhost:8080/smartcampus-api/api/v1/sensors?type=Temperature"
```

**7. Add a Sensor Reading**
```bash
curl -X POST http://localhost:8080/smartcampus-api/api/v1/sensors/TEMP-001/readings \
  -H "Content-Type: application/json" \
  -d "{\"value\":25.6}"
```

**8. Get Reading History**
```bash
curl -X GET http://localhost:8080/smartcampus-api/api/v1/sensors/TEMP-001/readings
```

**9. Delete Room with Sensors - expect 409 Conflict**
```bash
curl -X DELETE http://localhost:8080/smartcampus-api/api/v1/rooms/LIB-301
```

**10. Create Sensor with invalid roomId - expect 422**
```bash
curl -X POST http://localhost:8080/smartcampus-api/api/v1/sensors \
  -H "Content-Type: application/json" \
  -d "{\"id\":\"TEMP-002\",\"type\":\"Temperature\",\"status\":\"ACTIVE\",\"currentValue\":20,\"roomId\":\"INVALID\"}"
```

Part 1: Service Architecture & Setup
1.1 Resource lifecycle

By default, a JAX-RS resource class is typically created per request, meaning a fresh resource instance is used for each incoming HTTP request. This helps keep resource classes stateless and avoids accidental sharing of request-specific data between clients. However, if application data is stored only inside resource objects, that data would disappear after each request finishes.

For that reason, my implementation keeps shared application data in a separate in-memory store using shared collections rather than storing it in individual resource instances. Because multiple requests may access those collections at the same time, synchronization and thread safety are important. To reduce race conditions and inconsistent updates, shared maps should be handled carefully, for example with thread-safe collections such as ConcurrentHashMap. This design preserves data across requests while keeping the resource classes simple and safe.

1.2 Hypermedia / HATEOAS

Hypermedia is considered a hallmark of advanced RESTful design because the server includes navigational links in responses so that clients can discover available resources and actions dynamically. Instead of depending entirely on hardcoded endpoint knowledge, the client can follow links returned by the API.

This benefits client developers because it improves discoverability and makes the API easier to use and evolve. If endpoints change or expand, clients can rely on the links provided by the server rather than only on static external documentation. In my discovery endpoint, the response includes links to key collections such as rooms and sensors, which helps clients understand how to navigate the API from the root entry point.

Part 2: Room Management
2.1 Returning IDs only vs full room objects

Returning only room IDs reduces response size and saves network bandwidth, which can be useful when the client only needs identifiers. However, it often forces the client to make additional requests to fetch the full details of each room, which increases the number of round trips and complicates client-side logic.

Returning full room objects increases the response payload, but it is more convenient for most clients because the metadata is available immediately. This reduces extra requests and simplifies the user interface or consumer application. In my implementation, returning full room objects is more practical because it gives complete room information directly and makes the API easier to test and use.

2.2 Is DELETE idempotent?

Yes, the DELETE operation is idempotent. If a room exists and has no assigned sensors, the first DELETE request removes it from the system. If the same DELETE request is sent again, the room is already gone, so the server returns a not found response. Although the response code may differ between requests, the important point is that repeating the same request does not produce additional state changes after the first successful deletion.

Therefore, the operation is idempotent because multiple identical DELETE requests leave the server in the same final state: the room is absent. This matches the REST meaning of idempotency.

Part 3: Sensor Operations & Linking
3.1 @Consumes(MediaType.APPLICATION_JSON)

The @Consumes(MediaType.APPLICATION_JSON) annotation tells JAX-RS that the method only accepts request bodies sent as JSON. If a client sends data using another content type such as text/plain or application/xml, JAX-RS tries to find a resource method that can consume that format. If no suitable method exists, the framework rejects the request.

In practice, this usually results in HTTP 415 Unsupported Media Type. This is useful because it enforces a clear contract between client and server. It prevents the server from trying to interpret unsupported formats and ensures that clients send data in the expected JSON structure.

3.2 Why @QueryParam is better for filtering

Using @QueryParam is better for filtering because the client is still requesting the same collection resource, just with optional conditions applied. For example, /api/v1/sensors?type=CO2 clearly means “return the sensors collection filtered by type.” This fits REST conventions for filtering, sorting, searching, and pagination.

If the filter is placed in the path, such as /api/v1/sensors/type/CO2, the URL structure becomes more rigid and less flexible. It also becomes harder to combine multiple optional filters later. Query parameters are easier to extend, for example /api/v1/sensors?type=CO2&status=ACTIVE. For that reason, query parameters are generally more suitable for filtering and searching collection resources.

Part 4: Deep Nesting with Sub-Resources
4.1 Benefits of the Sub-Resource Locator pattern

The Sub-Resource Locator pattern improves code organisation by separating responsibilities into smaller resource classes. In this API, SensorResource handles sensor operations, while SensorReadingResource handles the readings that belong to a specific sensor. This makes the design more modular and easier to understand.

If every nested path were implemented in one very large controller class, the code would become harder to read, maintain, test, and extend. Delegating nested logic to a dedicated sub-resource reduces complexity and matches the real structure of the domain, where readings are naturally children of a specific sensor. This is especially useful as APIs grow larger.

4.2 Updating currentValue after posting a reading

When a new reading is posted successfully, the API stores that reading in the sensor’s history and also updates the currentValue field of the parent sensor. This keeps the summary information and the historical data consistent.

This is important because some clients may request the sensor itself, while others may request the full reading history. If currentValue were not updated when a reading was added, the API could return inconsistent results. Updating both pieces of information as part of the same operation ensures better data consistency across the system.

Part 5: Advanced Error Handling, Exception Mapping & Logging
5.1 Why use 409 Conflict for deleting a room with sensors

409 Conflict is appropriate because the request is valid in form, but it conflicts with the current state of the resource. In this case, a room cannot be deleted while sensors are still assigned to it, because doing so would create orphaned sensor data.

In my implementation, the API throws a custom RoomNotEmptyException, and an exception mapper converts it into an HTTP 409 Conflict response with a JSON body explaining the problem. This gives the client a clear and meaningful error instead of a generic failure.

5.2 Why 422 is more accurate than 404 for a missing linked room

HTTP 422 Unprocessable Entity is more semantically accurate because the request reaches a valid endpoint and the JSON body itself is syntactically correct, but the submitted data cannot be processed according to business rules. In this scenario, the sensor request contains a roomId that does not exist.

A 404 Not Found would usually describe a missing endpoint or missing target resource in the URL itself. Here, the problem is inside the payload, not in the request path. Therefore, 422 better communicates that the request format was understood, but its content was invalid for processing.

5.3 Why use 403 Forbidden for maintenance mode

403 Forbidden is appropriate because the sensor exists and the client’s request is understood, but the requested action is not allowed in the sensor’s current state. A sensor marked MAINTENANCE should not accept new readings until it becomes available again.

This makes 403 a good choice because the problem is not a malformed request and not a missing resource. Instead, the server is deliberately refusing the operation because of a state-based rule.

5.4 Risks of exposing Java stack traces

Exposing internal Java stack traces to external users is risky because they reveal details about the internal implementation of the system. A stack trace may expose package names, class names, method names, framework details, server file paths, and the exact location of internal failures.

An attacker could use this information to learn the application structure, identify technologies in use, search for known vulnerabilities, and target weak points more effectively. For this reason, the API should return only a generic 500 Internal Server Error message to the client while logging the full technical error only on the server side.

5.5 Why use filters for logging

JAX-RS filters are better for logging because logging is a cross-cutting concern that applies to many endpoints in the same way. By using a request filter and a response filter, the API can centrally log the HTTP method, URI, and final response status for every request.

This avoids repeating Logger.info() calls in every resource method, reduces duplicated code, and keeps resource classes focused on business logic. It also makes future logging changes easier, because the behaviour can be updated in one place instead of throughout the whole project.ws	
