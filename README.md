# Emergency Dispatch System  
FE405 – Designing Apps and APIs

## Summary
This project is an emergency reporting and dispatch system. Citizens submit incidents, dispatchers manage those incidents, and responders receive assignments and update the status. 

## User roles
Citizen: submits reports  
Dispatcher: views reports and assigns responders  
Responder: receives assignments and updates progress  

---

# Project elements

### System components
1. Citizen app or web form for creating reports
2. Dispatcher dashboard with incident list and assignment controls
3. Responder interface for receiving tasks and sending status updates
4. Backend service for incidents, users, assignments, and notifications
5. Database for storing reports, users, and status history

---

# Architecture

### Patterns
The MVP will start as one backend service rather than many. This avoids early complexity.  
If needed, notifications or incident handling could be separated later.

### Communication
Frontends talk to the backend over HTTP.  
The backend reads and writes to the database.  
Notifications may run as background jobs so they do not block requests.  
Status updates can be pulled periodically or streamed later if needed.

---

# AuthN and AuthZ

AuthN:  
All users authenticate with basic login.

AuthZ:  
Permissions depend on role.  
- Citizens can only submit and view their own reports  
- Dispatchers can view all reports and assign responders  
- Responders can only view incidents assigned to them and update status  

---
# User Flows

### This is the example flow chart of a citizen making a report:

```mermaid
sequenceDiagram
    actor Citizen
    Citizen ->> AuthService: Authenticate user
    AuthService -->> Citizen: Token

    Citizen ->> IncidentService: report
    IncidentService ->> RedisDB: Store report in hot cache
    RedisDB -->> IncidentService: OK

    IncidentService ->> NotificationService: Notify dispatchers of new report
    NotificationService -->> DispatcherDashboard: Push update

    IncidentService -->> Citizen: Report created (ID + status)
```


### This is the example flow chart of the dispatcher receiving the report and assigning a responder:
```mermaid
sequenceDiagram
    actor Dispatcher
    participant Dashboard
    participant AuthService
    participant RedisDB
    participant NotificationService
    participant ResponderApp

    Dispatcher ->> AuthService: Login
    AuthService -->> Dispatcher: Token

    Dispatcher ->> Dashboard: Open incident list
    Dashboard <<->> RedisDB: Websocket incidents


    Dispatcher ->> RedisDB: Dispatcher assigns reponder
    RedisDB ->> NotificationService: Send assignment notification

    NotificationService ->> ResponderApp: Push/SMS/Telegram message
    RedisDB -->> Dashboard: Assignment confirmed
```


### This is the example flow chart of the reponder receiving their report from dispatch
```mermaid
sequenceDiagram
    actor Responder
    participant ResponderApp
    participant AuthService
    participant AssignmentHandler
    participant RedisDB
    participant NotificationService
    participant DispatcherDashboard
    participant CitizenApp

    Responder ->> AuthService: Login
    AuthService -->> Responder: Token

    NotificationService ->> ResponderApp: New assignment received

    Responder ->> AssignmentHandler: Accept or Decline assignment
    AssignmentHandler ->> RedisDB: Update incident status in hot store
    RedisDB -->> AssignmentHandler: OK

    AssignmentHandler ->> NotificationService: Dispatch status update event

    NotificationService ->> DispatcherDashboard: Push updated incident status
    NotificationService ->> CitizenApp: Push updated incident status
    NotificationService -->> ResponderApp: Status acknowledged
```


### This is the example flow chart of how the system would use Redis as a quick and temp system for reports, which after a few seconds would get moved to the postgreSQL DB for long term storage
```mermaid
sequenceDiagram
    participant RedisDB
    participant BackgroundWorker
    participant PostgreSQL
    participant S3

    loop Every few seconds
        BackgroundWorker ->> RedisDB: Scan for new/unprocessed reports
        RedisDB -->> BackgroundWorker: Return report data

        alt Report contains media
            BackgroundWorker ->> S3: Upload media files
            S3 -->> BackgroundWorker: Return media URLs
        end

        BackgroundWorker ->> PostgreSQL: INSERT report (with media URLs if any)
        PostgreSQL -->> BackgroundWorker: Insert OK

        BackgroundWorker ->> RedisDB: Mark report as processed OR remove from Redis
        RedisDB -->> BackgroundWorker: OK
    end
```