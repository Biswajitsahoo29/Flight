# Flight Booking Project: End-to-End Working Explanation

This file explains how your project  works from frontend to backend, how each part communicates, why each service exists, and what happens during the main user flows.

It is based on the current code inside this repository on July 23, 2026.

## 1. Big Picture

Your project is a microservice-based flight booking system with:

- a React + Vite frontend in `frontend/`
- six Spring Boot backend microservices
- one MySQL database per service
- JWT authentication for protected APIs
- OpenFeign for backend-to-backend HTTP communication

The idea is:

1. The frontend shows screens for login, registration, flight search, booking, history, profile, and admin management.
2. The frontend sends REST API calls to different backend services depending on the feature.
3. Some backend services work alone.
4. Some backend services call other backend services to complete a request.
5. Each service stores only its own data in its own database.

That separation is the main architectural idea of this project.

## 2. Folder-Level Architecture

### Frontend

- `frontend/` contains the React application.
- It uses:
  - `react`
  - `react-router-dom`
  - `redux-toolkit`
  - `react-redux`
  - `axios`
  - `vite`

### Backend services

- `flight-service` on port `8081`
- `user-service` on port `8082`
- `search-service` on port `8083`
- `booking-service` on port `8084`
- `seat-service` on port `8085`
- `ticket-service` on port `8086`

### Databases

Each service uses its own MySQL schema:

- `flight_service_db`
- `user_service_db`
- `search_service_db`
- `booking_service_db`
- `seat_service_db`
- `ticket_service_db`

This is the database-per-service pattern.

## 3. Why the Project Is Split This Way

Instead of one big backend, the project is divided by responsibility:

- `flight-service`: owns flight master data
- `user-service`: owns users, login, registration, profile, and user booking history aggregation
- `search-service`: owns search requests and search history
- `seat-service`: owns seat inventory and seat reservation state
- `booking-service`: owns booking orchestration and booking records
- `ticket-service`: owns generated tickets and PDF download

This helps because:

- each service has a clear job
- logic stays modular
- services can be changed independently
- one service does not directly touch another service's database
- cross-service communication happens through APIs only

## 4. Frontend Architecture

The frontend is the user-facing layer and works as the API consumer for all services.

### Main frontend files

- `frontend/src/main.jsx`
  - starts React
  - wraps the app with `Provider` and `BrowserRouter`
- `frontend/src/App.jsx`
  - global app shell
  - loads live data
  - shows notices and toasts
- `frontend/src/routes/AppRoutes.jsx`
  - defines public, protected, and admin-only routes
- `frontend/src/api/client.js`
  - central Axios client
  - injects JWT token into requests
- `frontend/src/api/index.js`
  - defines all API helper methods for each backend service
- `frontend/src/redux/authSlice.js`
  - stores login session in Redux and `localStorage`
- `frontend/src/redux/dataSlice.js`
  - stores loaded backend data like flights, users, seats, bookings, tickets

### Frontend routes

The app currently exposes these main pages:

- `/login`
- `/register`
- `/`
- `/booking`
- `/profile`
- `/bookings`
- `/my-bookings`
- `/manage/*`

### Route protection

There are two important route guards:

- `ProtectedRoute`
  - user must have a token
  - otherwise redirected to login
- `AdminRoute`
  - user role must be `ROLE_ADMIN` or `ADMIN`
  - otherwise redirected to `/`

So the frontend already enforces who can see the booking area and who can access management tools.

## 5. How the Frontend Talks to the Backend

The frontend uses one shared Axios client in `frontend/src/api/client.js`.

### Base URLs

It builds base URLs from environment variables or defaults:

- flights -> `http://localhost:8081`
- users -> `http://localhost:8082`
- search -> `http://localhost:8083`
- bookings -> `http://localhost:8084`
- seats -> `http://localhost:8085`
- tickets -> `http://localhost:8086`

### JWT handling

Before every request:

- Axios reads `aeroswift.token` from `localStorage`
- if a token exists, it sends:
  - `Authorization: Bearer <token>`

If a response comes back with `401`:

- frontend removes the saved token
- frontend removes the saved session

That keeps the frontend and backend auth states aligned.

### Important note

The active runtime API layer is `frontend/src/api/*`.

There is also a `frontend/src/services/` folder, but it does not appear to be used by the current React app flow. The live app is calling the backend through `src/api/client.js` and `src/api/index.js`.

## 6. Authentication Flow

Authentication is handled by `user-service`.

### Registration flow

1. User fills the registration form in `RegisterPage.jsx`.
2. Frontend sends `POST /auth/register` to `user-service`.
3. `AuthController` receives the request.
4. `AuthServiceImpl.register()`:
   - checks whether email already exists
   - calls `userService.createUser(...)`
   - user password is hashed with `BCryptPasswordEncoder`
   - default role becomes `ROLE_USER`
5. After saving the user, `user-service` generates a JWT.
6. Frontend stores the token and session in Redux and `localStorage`.
7. User is redirected into the app.

### Login flow

1. User fills email and password in `LoginPage.jsx`.
2. Frontend sends `POST /auth/login`.
3. `AuthServiceImpl.login()`:
   - finds the user by email
   - checks the raw password against the stored BCrypt hash
   - generates JWT with email as subject and role in token claims
4. Frontend saves the session using `authSlice`.

### Why JWT is used here

JWT is used so the backend remains stateless:

- no server-side session storage is needed
- each request contains the identity token
- every service can validate the token independently

## 7. Security Design Across Services

All services use:

- Spring Security
- a custom `JwtAuthenticationFilter`
- stateless session management
- CORS allowing the Vite frontend

### How JWT validation works

For protected endpoints:

1. Request reaches the service.
2. `JwtAuthenticationFilter` reads the `Authorization` header.
3. It checks the token using the local `JwtService`.
4. It extracts:
   - subject
   - role
5. It puts the authenticated user into Spring Security context.

Important design detail:

- all services use the same JWT secret value pattern
- because of that, a token issued by `user-service` can be validated by the other services too

### Endpoint access style

Examples from the current code:

- `user-service`
  - `/auth/**` is public
  - `/users/me` is for logged-in users and admins
  - `/users` admin-only for full list/create/update/delete
- `flight-service`
  - `GET /flights/**` is public
  - flight creation/update/delete is admin-only
- `search-service`
  - `GET /search` is public
- `seat-service`
  - viewing/reserving/releasing seats requires authenticated user or admin
  - seat create/update/delete is admin-only
- `booking-service`
  - `POST /bookings` requires `ROLE_USER`
  - full list is admin-only
  - booking lookup/update/cancel/delete requires authenticated user or admin
- `ticket-service`
  - create/generate requires `ROLE_USER`
  - read/update requires authenticated user or admin
  - delete is admin-only

This means the frontend route protection and backend role protection work together.

## 8. Backend Internal Pattern Used in Every Service

Each Spring Boot service follows the same layered pattern:

- `controller`
  - HTTP endpoint entry point
- `service`
  - business interface
- `service/impl`
  - actual business logic
- `repository`
  - database access with Spring Data JPA
- `entity`
  - table model
- `dto`
  - request/response objects
- `exception`
  - error classes and global handlers
- `client`
  - OpenFeign interfaces for calling other services

This makes every service consistent and easier to understand.

## 9. Data Owned by Each Service

### `flight-service`

Stores:

- flight number
- airline
- route
- departure and arrival date/time
- duration
- price
- available seats
- total seats
- flight status
- aircraft type
- terminal
- gate

### `user-service`

Stores:

- first name
- last name
- email
- phone number
- password hash
- date of birth
- gender
- nationality
- passport number
- address
- role

### `search-service`

Stores:

- source
- destination
- departure date
- return date
- passenger count
- travel class
- search timestamp

### `seat-service`

Stores:

- flight ID
- seat number
- seat type
- seat class
- price
- status

### `booking-service`

Stores:

- booking reference
- user ID
- flight ID
- seat ID
- booking date
- booking status
- payment status
- total fare

### `ticket-service`

Stores:

- ticket number
- booking ID
- flight ID
- user ID
- seat number
- issue date
- ticket status

## 10. Backend-to-Backend Communication

The services communicate through HTTP using OpenFeign.

### Why OpenFeign is used

Feign makes service-to-service calls look like Java method calls instead of manually building HTTP requests each time.

Example:

- `search-service` has a `FlightClient`
- `booking-service` has `UserClient`, `FlightClient`, `SeatClient`, `TicketClient`
- `seat-service` has a `FlightClient`
- `ticket-service` has `BookingClient`, `FlightClient`, `UserClient`
- `user-service` has `BookingClient`, `FlightClient`, `TicketClient`

### Feign base URLs

These come from `application.properties`, for example:

- `flight.service.url`
- `user.service.url`
- `booking.service.url`
- `seat.service.url`
- `ticket.service.url`

So each service knows where to call the others.

## 11. Feign Authentication Propagation

A very important part of your design is the `FeignAuthInterceptor`.

It exists in services that call protected downstream services, such as:

- `booking-service`
- `user-service`
- `ticket-service`
- `seat-service`

What it does:

1. Reads the incoming request's `Authorization` header.
2. Copies that same header into the outgoing Feign request.

Why this matters:

- when `booking-service` calls `seat-service`, the seat service still receives the original user's JWT
- downstream services can apply their own security rules using the same identity

This is one of the most important communication details in the project.

Special case:

- `search-service` does not need this for flight search because it calls a public `GET /flights/search` endpoint

## 12. Core User Flow: App Startup and Data Loading

When the frontend starts:

1. `main.jsx` mounts the app with Redux and Router.
2. `App.jsx` reads auth state from Redux.
3. If token exists, or the user is on `/`, it calls `refreshData()`.

### What `refreshData()` loads

If the current user is admin:

- all flights
- all users
- all seats
- all bookings
- all tickets

If the current user is not admin:

- all flights
- only the current user info
- no admin datasets for seats/bookings/tickets list

This is a nice design because:

- the main dashboard can stay hydrated
- admins get management data
- regular users do not unnecessarily load admin collections

## 13. Core User Flow: Flight Search

This flow starts in `BookingPage.jsx`.

### Frontend side

The search form collects:

- source
- destination
- departure date
- return date
- number of passengers
- travel class

When submitted:

- frontend calls `searchApi.searchFlights(...)`
- this maps to `GET /search?...` on `search-service`

### Backend side

`search-service` does the following:

1. `SearchController` receives query parameters into `SearchRequestDto`.
2. `SearchServiceImpl.validateSearchRequest()` checks:
   - source and destination must not match
   - return date cannot be before departure date
3. Search request is saved into `search_service_db` as `SearchHistory`.
4. `search-service` calls `flight-service` through Feign:
   - `GET /flights/search`
5. `flight-service` searches its own database using:
   - source
   - destination
   - departure date
   - `availableSeats >= numberOfPassengers`
6. Matching flights return to `search-service`.
7. `search-service` maps them into `SearchResponseDto`.
8. Frontend shows the flight cards.

### Why there is a separate `search-service`

This service does two jobs:

- keeps search logic separate from flight CRUD
- records user search history independently

That is useful because searching is a different business concern from managing flight records.

### Current limitation

The frontend captures `travelClass` and `returnDate`, and `search-service` stores them, but the actual flight filtering currently depends on `flight-service`, which only filters by:

- source
- destination
- departure date
- passenger count through available seats

So `travelClass` and return-trip logic are not fully enforced in the flight query yet.

## 14. Core User Flow: Flight Selection and Seat Loading

After a user selects a flight:

1. `BookingPage.jsx` calls `seatApi.byFlight(flight.id)`.
2. Frontend sends `GET /seats/flight/{flightId}` to `seat-service`.
3. `seat-service` loads all seats for that flight from `seat_service_db`.
4. Frontend filters them to only seats where `status === "AVAILABLE"`.
5. UI shows a seat map-like selection panel.

Why `seat-service` is separate:

- flights own route/schedule data
- seats own seat-level inventory and reservation state

That keeps seat locking logic isolated.

## 15. Core User Flow: Booking Creation

This is the most important backend orchestration flow in the system.

### Frontend side

When the user confirms booking, `BookingPage.jsx` sends:

- `userId`
- `flightId`
- `seatId`
- `bookingStatus`
- `paymentStatus`
- `totalFare`

to:

- `POST /bookings`

### Backend side in `booking-service`

`BookingServiceImpl.createBooking()` does this in order:

1. normalizes and validates booking values
2. verifies the user exists by calling `user-service`
3. verifies the flight exists by calling `flight-service`
4. loads the seat from `seat-service`
5. checks the seat belongs to the selected flight
6. checks the seat is currently `AVAILABLE`
7. calls `seat-service` to reserve the seat
8. creates a booking reference like `BKG-XXXXXXXXXX`
9. saves the booking in `booking_service_db`
10. returns booking data including `seatNumber`

### Why this orchestration is correct

`booking-service` is the right place for this because a booking depends on multiple domains:

- user
- flight
- seat

No single domain service should own the full booking transaction except `booking-service`.

### Failure handling

If the seat is reserved but booking save fails:

- `booking-service` attempts to release the seat again

That is a very useful compensation step for microservice workflows.

## 16. What Happens Inside Seat Reservation

When `booking-service` calls `seat-service` to reserve a seat:

1. `seat-service` loads the seat by ID
2. checks it is not already reserved
3. updates seat status to `RESERVED`
4. saves the seat
5. calls `flight-service` to decrease the parent flight's `availableSeats`

So one booking action changes data in two places:

- seat status in `seat_service_db`
- flight seat count in `flight_service_db`

### Why this exists

The detailed seat record and the summarized flight availability are both useful:

- seat record is needed for exact seat assignment
- flight summary count is needed for quick flight searching

This is why `seat-service` also synchronizes the count in `flight-service`.

## 17. Core User Flow: Ticket Generation

After a booking is created, the frontend can issue a ticket.

### Frontend side

`BookingPage.jsx` calls:

- `POST /tickets/generate`

with:

- `bookingId`
- `ticketStatus`

### Backend side in `ticket-service`

`TicketServiceImpl.generateTicket()` does this:

1. validates ticket status
2. calls `booking-service` to verify the booking exists
3. checks booking is not cancelled
4. checks booking contains a seat number
5. generates a ticket number like `TKT-XXXXXXXXXX`
6. copies booking-linked values into ticket:
   - bookingId
   - flightId
   - userId
   - seatNumber
7. stores the ticket in `ticket_service_db`
8. returns ticket data

### Why `ticket-service` is separate

Ticket issuance is a separate business artifact from booking:

- booking means reservation exists
- ticket means travel document was issued

That separation is realistic and makes cancellation/download flows cleaner.

## 18. Core User Flow: Ticket PDF Download

When the frontend downloads a ticket:

1. it calls `GET /tickets/{id}/download`
2. `ticket-service` loads:
   - ticket from its own DB
   - booking from `booking-service`
   - flight from `flight-service`
   - user from `user-service`
3. it generates a PDF in memory
4. it returns PDF bytes to the frontend
5. frontend converts blob to a downloadable file

### Library used

The PDF is generated in Java using `com.lowagie.text`, which is the OpenPDF/iText-style document API.

### Why this flow is nice

The ticket PDF becomes a final composed document built from multiple services without any service sharing another service's database.

## 19. Core User Flow: Booking Cancellation

Cancellation is handled carefully.

When the frontend cancels a booking:

1. frontend calls `PUT /bookings/{id}/cancel`
2. `booking-service` marks the booking as `CANCELLED`
3. `booking-service` releases the reserved seat through `seat-service`
4. `seat-service` sets seat status back to `AVAILABLE`
5. `seat-service` calls `flight-service` to increase `availableSeats`
6. `booking-service` also tries to cancel the ticket in `ticket-service`

Important detail:

- if no ticket exists yet, cancellation still succeeds
- that is handled safely in code

So one cancel action reverses the booking side effects.

## 20. Core User Flow: User Profile and History

### Profile

`ProfilePage.jsx` uses:

- `GET /users/me`
- `PUT /users/me`

This keeps profile editing inside `user-service`.

### Booking history

Booking history is more interesting because `user-service` acts as an aggregator.

When the frontend opens booking history:

1. it calls `GET /users/me/bookings`
2. `user-service` identifies the current user from Spring Security authentication
3. `user-service` finds the user by email
4. it calls `booking-service` for that user's bookings
5. for each booking, it also loads:
   - flight details from `flight-service`
   - ticket details from `ticket-service` if available
6. it builds `UserBookingHistoryDto`
7. frontend displays booking + flight + ticket in one page

### Why this design is useful

The frontend gets one rich response instead of manually calling:

- user
- bookings
- flights
- tickets

for every history card.

So `user-service` becomes the composition layer for user-centric history.

## 21. Admin Management Flow

The management page is your internal admin console.

It can:

- create/update/delete flights
- create/update/delete users
- create/update/delete seats
- update bookings
- update/cancel tickets
- look up records by ID
- run direct flight search
- load bookings by user
- generate tickets directly
- reserve or release seats manually

### Why this page matters

It acts like:

- a lightweight admin panel
- a service integration test UI
- a manual seeding and debugging tool

That is very useful in a microservice project because it lets you verify backend behavior without Postman every time.

## 22. Error Handling

Every service has:

- custom exception classes
- `GlobalExceptionHandler`

So exceptions become structured HTTP responses instead of random stack traces.

On the frontend:

- Axios response interceptor extracts backend error messages
- validation-style error maps are flattened into readable strings
- pages show those messages through notices and toast messages

This is why the app can show clear errors like:

- invalid login
- flight not found
- seat already reserved
- unable to reach another service

## 23. Important Technical Choices Used in This Project

### Backend technologies

- Java
- Spring Boot
- Spring Web
- Spring Security
- Spring Data JPA
- Hibernate
- MySQL
- OpenFeign
- JWT
- OpenPDF/iText-style PDF generation

### Frontend technologies

- React
- Vite
- React Router
- Redux Toolkit
- React Redux
- Axios
- Tailwind dependency is installed, but the app styling currently looks mostly custom/CSS-driven rather than Tailwind-centric in the main flow

## 24. Service-by-Service Communication Summary

Here is the cleanest way to remember the communication map.

### Frontend -> services

The frontend directly calls:

- `user-service`
- `search-service`
- `flight-service`
- `seat-service`
- `booking-service`
- `ticket-service`

### Service -> service

- `search-service` -> `flight-service`
- `booking-service` -> `user-service`
- `booking-service` -> `flight-service`
- `booking-service` -> `seat-service`
- `booking-service` -> `ticket-service` during cancellation
- `seat-service` -> `flight-service`
- `ticket-service` -> `booking-service`
- `ticket-service` -> `flight-service`
- `ticket-service` -> `user-service`
- `user-service` -> `booking-service`
- `user-service` -> `flight-service`
- `user-service` -> `ticket-service`

## 25. The Most Important Runtime Flows in One Line Each

- Register: frontend -> `user-service` -> create user -> return JWT
- Login: frontend -> `user-service` -> validate password -> return JWT
- Search: frontend -> `search-service` -> save history -> `flight-service`
- Load seats: frontend -> `seat-service`
- Book: frontend -> `booking-service` -> `user-service` + `flight-service` + `seat-service`
- Reserve seat: `booking-service` -> `seat-service` -> `flight-service`
- Generate ticket: frontend -> `ticket-service` -> `booking-service`
- Download PDF: frontend -> `ticket-service` -> `booking-service` + `flight-service` + `user-service`
- View history: frontend -> `user-service` -> `booking-service` + `flight-service` + `ticket-service`
- Cancel booking: frontend -> `booking-service` -> `seat-service` + `ticket-service`

## 26. Practical Strengths of Your Current Design

- Clear domain separation
- Good use of database-per-service architecture
- JWT auth is implemented across services, not just at login
- Feign auth propagation is a strong touch and makes protected service chains work correctly
- Booking cancellation compensates related resources
- Seat reservation also synchronizes flight availability count
- User booking history is nicely aggregated
- Ticket PDF generation gives the project a more complete end-to-end feel
- The frontend management page is genuinely useful for testing the whole system

## 27. Practical Limitations or Things to Be Aware Of

These are not necessarily mistakes, but they are good to understand.

### Search is only partially rich

`travelClass` and `returnDate` are collected and stored, but not fully used in flight matching yet.

### Distributed transaction behavior is manual

You are using compensation logic like seat release after failure, which is good, but this is still not a full distributed transaction system.

### Search endpoint is public

This is fine for many systems, but it is a design choice worth remembering.

### Flight GET endpoints are public

That means anyone can list or search flights without login.

### Frontend has an unused `src/services/` layer

The live app uses `src/api/` instead, so that extra folder may be historical or experimental.

## 28. If You Want to Explain This in an Interview

A simple interview version would be:

"This project is a React frontend with multiple Spring Boot microservices. The frontend talks directly to separate services for users, flights, search, seats, bookings, and tickets. Authentication is handled with JWT by the user service, and the token is reused across the other services. Booking creation is orchestrated by the booking service, which verifies the user, flight, and seat through Feign clients, reserves the seat, and stores the booking. Seat reservation updates both seat state and the flight's available seat count. Ticket generation is handled by a dedicated ticket service that validates the booking before issuing a ticket and generating a downloadable PDF. Each service has its own database, so services communicate only through REST APIs, not by sharing tables."

## 29. Final Mental Model

The easiest way to think about your system is this:

- `user-service` knows who the passenger is
- `flight-service` knows what flights exist
- `search-service` knows what the user searched for
- `seat-service` knows which exact seats are available or reserved
- `booking-service` turns a chosen user + flight + seat into a real booking
- `ticket-service` turns a booking into a final travel ticket
- the frontend ties all of that together into one usable booking experience

That is the full working model of your project from frontend to backend.
