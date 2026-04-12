```mermaid
C4Context
    title Test
    Person_Ext(user, "User", "A user")
    System_Ext(system, "System", "A system")
    Rel(user, system, "HTTPS: User sends request to system", "HTTPS")
    Rel_Back(user, system, "HTTPS: System sends response to user")
    linkStyle 1 stroke-dasharray: 5 5
    UpdateLayoutConfig($c4ShapeInRow="3", $c4BoundaryInRow="1")
```