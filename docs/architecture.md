## Product choice

Yandex GO
Link: <https://go.yandex>
"Яндекс Go"- мобильное приложение компании «Яндекс», связанное с транспортировкой и доставкой. Было создано на базе «Яндекс.Такси».

## Main components

![Yandex Go Component Diagram](./diagrams/out/yandex-go/architecture-component/Component Diagram.svg)
![Yandex Go Diagram code](./diagrams/src/yandex-go/architecture-component.puml)

1. API Gateway
Acts as the single entry point for all client requests, handling routing, authentication, and request aggregation before forwarding calls to internal services.
2. User Service
Manages user-related data and logic, such as profiles, authentication details, preferences, and user status.
3. Dispatch Service
Responsible for matching riders with drivers by processing availability, location, and demand in real time.
4. Pricing Service
Calculates trip prices dynamically based on distance, time, demand, and surge pricing rules.
5. Payment Service
Handles payment workflows, including charging users, interacting with payment providers, and tracking transaction status.

## Data flow

![Yandex Go Data Flow Diagram](./diagrams/out/yandex-go/architecture-sequence/Sequence Diagram.svg)
![Yandex Go Data flow code](./diagrams/src/yandex-go/architecture-secuence.puml)

Booking & Async Dispatch

Components involved and how they interact

Mobile App → API Gateway
    The app sends createOrder(class, payment_id) containing the selected ride class and chosen payment method.
    This is a synchronous RPC call initiating the booking.

API Gateway → Dispatch Service
    The gateway forwards the request to the Dispatch Service to start ride creation and driver matching.
    Data passed: ride class, payment identifier, user context.

Dispatch Service → User Service
    The Dispatch Service validates that the selected payment method is valid and usable.
    Data exchanged: payment method ID and validation result (OK).

Dispatch Service → Operational DB
    A new ride order is persisted with status SEARCHING.
    Data stored: order ID, user ID, pickup/dropoff, status.

Dispatch Service → State Cache
    Performs a geospatial lookup to find nearby available drivers.
    Data exchanged: pickup location → list of candidate drivers.

Dispatch Service (internal logic)
    Runs the matching algorithm to select the best driver based on distance, availability, and other heuristics.

Dispatch Service → Operational DB
    Updates the order status to ASSIGNED once a driver is chosen.

Dispatch Service → Kafka Event Bus
    Publishes a RideAssigned event.
    Event data: order ID, driver ID, assignment timestamp.

Kafka Event Bus → Push Service
    Push Service consumes the RideAssigned event asynchronously.

Push Service → Mobile App
    Sends a push notification (“Driver Found”) to the user.
    Data: driver info, order ID.

Mobile App → User
    Updates the UI to show the assigned driver on the map.

## Deployment

![Yandex Go Data Flow Diagram](./diagrams/out/yandex-go/architecture-deployment/Deployment Diagram.svg)
![Yandex Go Data flow code](./diagrams/src/yandex-go/architecture-deployment.puml)

Where the components are deployed

Client devices
The Yandex Go Mobile App runs on user smartphones (iOS/Android), and the Web App runs inside a web browser on user computers. Both communicate with backend services over HTTPS (REST APIs).

Cloud edge / ingress layer
A Load Balancer and API Gateway are deployed in Yandex Cloud and act as the public entry point, handling HTTPS traffic, authentication, and routing requests to backend services.

Application tier (Kubernetes cluster)
Core services (User, Dispatch, Pricing, Payment, Maps & Routing, Notification) are deployed as containerized pods inside a Kubernetes cluster and communicate internally using gRPC.

Data & messaging layer
Redis runs as a cache cluster for low-latency data access, Kafka runs as a message broker for event-driven communication, and YDB plus ClickHouse are deployed as distributed storage clusters for operational and analytics data.

External services
Yandex Pay API and Yandex Maps API are deployed outside the system boundary and are accessed securely over HTTPS/gRPC from backend services.

## Assumptions

I assume the pricing service handles surge pricing calculations based on demand and supply in real-time.

## Open questions

How does the actual load balancing mechanism work between the microservices in production?
