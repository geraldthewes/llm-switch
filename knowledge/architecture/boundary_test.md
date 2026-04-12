```mermaid
C4Context
    title Test
    System_Ext(s1, "System 1", "Desc")
    System_Ext(s2, "System 2", "Desc")
    Boundary(b, "Boundary", dashed) {
        s1
        s2
    }
    Rel(s1, s2, "Link", "HTTPS")
    UpdateLayoutConfig($c4ShapeInRow="3", $c4BoundaryInRow="1")
```