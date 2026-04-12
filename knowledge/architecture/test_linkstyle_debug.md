```mermaid
C4Context
    title Test
    Person_Ext(user, "User", "A user")
    System_Ext(system, "System", "A system")
    Rel(user, system, "First", "HTTPS")
    Rel(user, system, "Second", "HTTPS")
    linkStyle 1 stroke-dasharray: 5 5
    UpdateLayoutConfig($c4ShapeInRow="3", $c4BoundaryInRow="1")
```