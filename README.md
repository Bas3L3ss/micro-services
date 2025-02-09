# Improvement Plan for Ride-Hailing App

_This is a tutorial app that will be personalized later. I’ve learned a lot from this project and thoroughly retained all the necessary backend knowledge. Feeling good!_

---

## I. Core Functional Improvements

### 1. Enhance Ride Acceptance Information

- **Problem:** Currently, when a ride is accepted, the user does not receive any captain information.
- **Action:**
  - Update the `acceptRide` route to capture and store the captain's ID (or complete details) in the ride document.
  - Ensure that when the `ride-accepted` event is published, the ride data includes the captain information.

### 2. Improve Ride Lifecycle Management

- **Ride Cleanup:**
  - **Problem:** There is no mechanism for cleaning up rides that remain untouched or are never accepted.
  - **Action:** Implement a scheduled cleanup process to remove or mark stale rides.
- **Ride Completion Process:**
  - **Problem:** The current implementation lacks a process to mark a ride as completed.
  - **Action:** Develop endpoints and logic for transitioning a ride from "started" to "completed," ensuring that the ride lifecycle is fully managed.

### 3. Implement Geographical Constraints for Ride Requests

- **Problem:** Captains can receive and accept rides regardless of their current location.
- **Action:**
  - Limit the ride requests a captain receives to those within a specific geographical radius.
  - Use geolocation filtering in the ride query or via a middleware to ensure that only relevant rides are dispatched to available captains.
  - **Backend Code Idea:**  
    Use a MongoDB geospatial query:
    ```js
    db.captains.find({
      location: {
        $near: {
          $geometry: { type: "Point", coordinates: [userLng, userLat] },
          $maxDistance: 5000, // within a 5km radius
        },
      },
    });
    ```

### 4. Implement Ride Type and Captain Type Matching

- **Problem:** The system currently does not differentiate between different ride types (Car vs. Motorbike) and service tiers (Economy vs. Luxury), nor does it distinguish the captain’s vehicle capabilities.
- **Action:**

  - **Update Ride Schema:**  
    Add fields for `rideType` and `serviceTier` so that each ride explicitly specifies the type of vehicle requested and the desired service tier.

    ```js
    const rideSchema = new mongoose.Schema(
      {
        captain: {
          type: mongoose.Schema.Types.ObjectId,
        },
        user: {
          type: mongoose.Schema.Types.ObjectId,
          required: true,
        },
        pickup: {
          type: String,
          required: true,
        },
        destination: {
          type: String,
          required: true,
        },
        status: {
          type: String,
          enum: ["requested", "accepted", "started", "completed"],
          default: "requested",
        },
        rideType: {
          type: String,
          enum: ["Car", "Motorbike"],
          required: true,
        },
        serviceTier: {
          type: String,
          enum: ["Economy", "Luxury"],
          required: true,
        },
      },
      {
        timestamps: true,
      }
    );
    ```

  - **Update Captain Schema:**  
    Include vehicle information so that each captain is associated with a vehicle type and a service tier.  
    **For captains with a single vehicle:**

    ```js
    const captainSchema = new mongoose.Schema(
      {
        userId: {
          type: mongoose.Schema.Types.ObjectId,
          required: true,
          ref: "User",
        },
        vehicleType: {
          type: String,
          enum: ["Car", "Motorbike"],
          required: true,
        },
        serviceTier: {
          type: String,
          enum: ["Economy", "Luxury"],
          required: true,
        },
        vehicleDetails: {
          brand: String,
          model: String,
          licensePlate: String,
        },
        location: {
          type: {
            type: String,
            enum: ["Point"],
            default: "Point",
          },
          coordinates: {
            type: [Number], // [longitude, latitude]
            required: true,
          },
        },
      },
      {
        timestamps: true,
      }
    );
    ```

    **For captains who own multiple vehicles:**  
    Allow them to register multiple vehicles and select an active one when going online.

    ```js
    const captainSchema = new mongoose.Schema(
      {
        userId: {
          type: mongoose.Schema.Types.ObjectId,
          required: true,
          ref: "User",
        },
        vehicles: [
          {
            vehicleType: {
              type: String,
              enum: ["Car", "Motorbike"],
              required: true,
            },
            serviceTier: {
              type: String,
              enum: ["Economy", "Luxury"],
              required: true,
            },
            brand: String,
            model: String,
            licensePlate: String,
          },
        ],
        activeVehicle: {
          type: mongoose.Schema.Types.ObjectId, // Reference to one of their vehicles
          ref: "Vehicle",
        },
        location: {
          type: {
            type: String,
            enum: ["Point"],
            default: "Point",
          },
          coordinates: {
            type: [Number], // [longitude, latitude]
            required: true,
          },
        },
      },
      {
        timestamps: true,
      }
    );
    ```

  - **Matching Logic:**  
    Ensure that when a ride request is made, the system only dispatches the ride to captains with matching `vehicleType` and `serviceTier`, and who are within the required geographic range.
    ```js
    const findMatchingCaptains = async (rideRequest) => {
      return await captainModel.find({
        "vehicles.vehicleType": rideRequest.rideType,
        "vehicles.serviceTier": rideRequest.serviceTier,
        "location.coordinates": {
          $near: {
            $geometry: {
              type: "Point",
              coordinates: rideRequest.pickupLocation,
            },
            $maxDistance: 5000, // Example: within 5km radius
          },
        },
      });
    };
    ```
    _Note:_ If using a single vehicle per captain, adjust the query accordingly.

### 5. Implement Real-Time Geolocation Data Capture and Tracking

- **Problem:** For accurate ride matching and dynamic updates, the system needs to capture the real-time location of users (when requesting rides) and captains (while online).
- **Action:**
  - **Front-End (Web App) Code Idea:**  
    Use the Geolocation API to get the user's current location:
    ```js
    navigator.geolocation.getCurrentPosition(
      (position) => {
        const { latitude, longitude } = position.coords;
        console.log(`User Location: ${latitude}, ${longitude}`);
        // Send these coordinates to your backend as part of the ride request
      },
      (error) => {
        console.error("Error getting location:", error);
      },
      {
        enableHighAccuracy: true,
        timeout: 5000,
        maximumAge: 0,
      }
    );
    ```
  - **Front-End (Native Mobile App):**  
    Use platform-specific libraries (e.g., `react-native-geolocation-service` for React Native or `geolocator` for Flutter) to capture the GPS coordinates.
  - **Backend Integration:**
    - When a ride is created, include the user's pickup coordinates.
    - Update the captain’s last known location periodically (via WebSockets or frequent API calls) so that geospatial queries remain accurate.
    - **Example Geospatial Query:** (already provided above)
      ```js
      db.captains.find({
        location: {
          $near: {
            $geometry: { type: "Point", coordinates: [userLng, userLat] },
            $maxDistance: 5000,
          },
        },
      });
      ```
  - **Real-Time Updates:**  
    Consider using WebSockets or services like Firebase Realtime Database to update captain locations dynamically rather than relying solely on stored coordinates.

### 6. Establish CI/CD and Testing Pipelines

- **Action:**
  - Set up continuous integration and continuous deployment pipelines.
  - Implement automated testing (unit tests, integration tests) for routes, services, and overall system reliability.
  - Ensure that new changes are automatically validated before deployment.

---

## II. Additional Enhancements & Refinements

### 1. Scalability & Performance Enhancements

- **Service Discovery:**
  - Consider integrating tools like Consul or Eureka to simplify inter-service communication and load balancing.
- **Database Optimization:**
  - If MongoDB becomes a bottleneck, explore sharding or partitioning strategies for high-load collections (e.g., rides, location data).
- **Real-Time Updates:**
  - Replace long polling with WebSockets for more efficient, real-time communication between services and clients.

### 2. Security & Authentication

- **OAuth2 Integration:**
  - Look into OAuth2 as an alternative for authenticating users and captains, especially if third-party login (Google, Facebook, Apple) is desired.
- **Role-Based Access Control (RBAC):**
  - Implement fine-grained permission systems as the app scales to handle multiple user roles.
- **Rate Limiting:**
  - Introduce rate limiting (e.g., using Redis) to protect against DDoS attacks and abuse by capping the number of requests per user/IP.

### 3. DevOps & Observability

- **Centralized Logging:**
  - Integrate logging solutions such as the ELK stack (Elasticsearch, Logstash, Kibana) or Prometheus with Grafana for real-time monitoring.
- **Distributed Tracing:**
  - Utilize tracing tools like OpenTelemetry or Jaeger to track requests across microservices, aiding in debugging and performance tuning.

### 4. Message Queue Optimization

- **Dead Letter Queue (DLQ):**
  - Configure DLQs in RabbitMQ to handle messages that cannot be processed successfully.
- **Retry Mechanism:**
  - Implement an exponential backoff strategy for retrying failed message deliveries to improve message processing resilience.

### 5. Potential Tech Stack Upgrades

- **Internal Communication:**
  - Evaluate using gRPC for internal service communication for improved performance and efficiency over REST.
- **Caching:**
  - Integrate Redis to cache frequently accessed data, such as captain locations and ride statuses, to reduce database load.
- **Relational Database Option:**
  - Consider using PostgreSQL for scenarios that require strong consistency, particularly for critical ride transaction data.

---

## Conclusion

By addressing the immediate issues—such as including captain details on ride acceptance, implementing ride cleanup and completion processes, applying geographical filters for ride requests, introducing ride type and captain type matching, and capturing real-time geolocation data for both users and captains—you will close the functional gaps in your current system. Additionally, incorporating the broader enhancements for scalability, security, observability, and potential tech stack upgrades will help ensure that your architecture is robust and ready for future growth.

This structured plan will not only improve the current functionality of your ride-hailing app but also pave the way for a more scalable, secure, and maintainable system. Happy coding!
