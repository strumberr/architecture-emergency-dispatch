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
