graph TD;
    actor Citizen
    Citizen ->> AuthService: Authenticate user
    AuthService -->> Citizen: Token

    Citizen ->> IncidentService: report
    IncidentService ->> RedisDB: Store report in hot cache
    RedisDB -->> IncidentService: OK

    IncidentService ->> NotificationService: Notify dispatchers of new report
    NotificationService -->> DispatcherDashboard: Push update

    IncidentService -->> Citizen: Report created (ID + status)
