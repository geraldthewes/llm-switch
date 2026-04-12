```mermaid
C4Context
    title Test
    Person_Ext(dev, "Developer", "A developer")
    System_Ext(consul, "Consul", "Consul")
    System_Ext(vault, "Vault", "Vault")
    Boundary(boundaryTrusted, "Trusted Zone (mTLS Required)", dashed) {
        consul
        vault
    }
    Rel(dev, consul, "Link", "HTTPS")
    UpdateLayoutConfig($c4ShapeInRow="3", $c4BoundaryInRow="1")
```