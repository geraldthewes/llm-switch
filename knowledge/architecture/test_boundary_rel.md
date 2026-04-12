```mermaid
C4Context
    title Test
    Person_Ext(user, "User", "A user")
    System_Ext(system, "System", "A system")
    Boundary(boundaryTrusted, "Trusted Zone", dashed) {
        system
    }
    Rel(user, system, "Link", "HTTPS")
    UpdateLayoutConfig($c4ShapeInRow="3", $c4BoundaryInRow="1")
```