```mermaid
C4Context
    title Test
    Person_Ext(user, "User", "A user")
    System_Ext(system, "System", "A system")
    Rel(user, system, "Uses", "HTTPS")
    UpdateLayoutConfig($c4ShapeInRow="3", $c4BoundaryInRow="1")
```