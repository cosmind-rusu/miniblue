# Database for MySQL

miniblue emulates Azure Database for MySQL Flexible Servers via ARM endpoints. This is a stub implementation -- it stores server and database metadata but does not connect to a real MySQL instance.

## API endpoints

### Flexible Servers

| Method | Path | Description |
|--------|------|-------------|
| `PUT` | `/subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.DBforMySQL/flexibleServers/{name}` | Create or update server |
| `GET` | `/subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.DBforMySQL/flexibleServers/{name}` | Get server |
| `DELETE` | `/subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.DBforMySQL/flexibleServers/{name}` | Delete server |
| `GET` | `/subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.DBforMySQL/flexibleServers` | List servers |

### Databases

| Method | Path | Description |
|--------|------|-------------|
| `PUT` | `.../flexibleServers/{server}/databases/{db}` | Create or update database |
| `GET` | `.../flexibleServers/{server}/databases/{db}` | Get database |
| `DELETE` | `.../flexibleServers/{server}/databases/{db}` | Delete database |
| `GET` | `.../flexibleServers/{server}/databases` | List databases |

## Create a flexible server

```bash
curl -X PUT "http://localhost:4566/subscriptions/sub1/resourceGroups/myRG/providers/Microsoft.DBforMySQL/flexibleServers/my-mysql" \
  -H "Content-Type: application/json" \
  -d '{
    "location": "eastus",
    "properties": {
      "administratorLogin": "myadmin"
    },
    "sku": {
      "name": "Standard_B1ms",
      "tier": "Burstable"
    }
  }'
```

Response (`201 Created`):

```json
{
  "id": "/subscriptions/sub1/resourceGroups/myRG/providers/Microsoft.DBforMySQL/flexibleServers/my-mysql",
  "name": "my-mysql",
  "type": "Microsoft.DBforMySQL/flexibleServers",
  "location": "eastus",
  "properties": {
    "fullyQualifiedDomainName": "localhost",
    "state": "Ready",
    "version": "8.0",
    "administratorLogin": "myadmin",
    "storage": { "storageSizeGB": 20 },
    "provisioningState": "Succeeded"
  },
  "sku": { "name": "Standard_B1ms", "tier": "Burstable" }
}
```

## Get a server

```bash
curl "http://localhost:4566/subscriptions/sub1/resourceGroups/myRG/providers/Microsoft.DBforMySQL/flexibleServers/my-mysql"
```

## List servers

```bash
curl "http://localhost:4566/subscriptions/sub1/resourceGroups/myRG/providers/Microsoft.DBforMySQL/flexibleServers"
```

## Delete a server

```bash
curl -X DELETE "http://localhost:4566/subscriptions/sub1/resourceGroups/myRG/providers/Microsoft.DBforMySQL/flexibleServers/my-mysql"
```

Returns `202 Accepted`. Deleting a server also removes all child database metadata.

## Create a database

The parent server must exist first.

```bash
curl -X PUT "http://localhost:4566/subscriptions/sub1/resourceGroups/myRG/providers/Microsoft.DBforMySQL/flexibleServers/my-mysql/databases/appdb" \
  -H "Content-Type: application/json" \
  -d '{
    "properties": {
      "charset": "utf8mb4",
      "collation": "utf8mb4_0900_ai_ci"
    }
  }'
```

Response (`201 Created`):

```json
{
  "id": "/subscriptions/sub1/resourceGroups/myRG/providers/Microsoft.DBforMySQL/flexibleServers/my-mysql/databases/appdb",
  "name": "appdb",
  "type": "Microsoft.DBforMySQL/flexibleServers/databases",
  "properties": {
    "charset": "utf8mb4",
    "collation": "utf8mb4_0900_ai_ci",
    "provisioningState": "Succeeded"
  }
}
```

## Get a database

```bash
curl "http://localhost:4566/subscriptions/sub1/resourceGroups/myRG/providers/Microsoft.DBforMySQL/flexibleServers/my-mysql/databases/appdb"
```

## List databases

```bash
curl "http://localhost:4566/subscriptions/sub1/resourceGroups/myRG/providers/Microsoft.DBforMySQL/flexibleServers/my-mysql/databases"
```

## Delete a database

```bash
curl -X DELETE "http://localhost:4566/subscriptions/sub1/resourceGroups/myRG/providers/Microsoft.DBforMySQL/flexibleServers/my-mysql/databases/appdb"
```

Returns `202 Accepted`.

## azlocal

```bash
# Use curl for MySQL ARM endpoints (azlocal does not yet have mysql commands)

# Create server
curl -X PUT "http://localhost:4566/subscriptions/sub1/resourceGroups/myRG/providers/Microsoft.DBforMySQL/flexibleServers/my-mysql" \
  -H "Content-Type: application/json" -d '{"location":"eastus"}'

# Create database
curl -X PUT "http://localhost:4566/subscriptions/sub1/resourceGroups/myRG/providers/Microsoft.DBforMySQL/flexibleServers/my-mysql/databases/mydb" \
  -H "Content-Type: application/json" -d '{}'

# Verify the resource group exists
azlocal group show --name myRG
```

## Full example

```bash
#!/bin/bash
set -e

BASE="http://localhost:4566/subscriptions/sub1/resourceGroups/myRG/providers/Microsoft.DBforMySQL/flexibleServers"

# Create resource group
curl -s -X PUT "http://localhost:4566/subscriptions/sub1/resourcegroups/myRG" \
  -H "Content-Type: application/json" \
  -d '{"location": "eastus"}'

# Create MySQL server
curl -s -X PUT "${BASE}/my-mysql" \
  -H "Content-Type: application/json" \
  -d '{"location": "eastus", "properties": {"administratorLogin": "admin"}}' | jq .name

# Create databases
curl -s -X PUT "${BASE}/my-mysql/databases/app_db" \
  -H "Content-Type: application/json" -d '{}' | jq .name
curl -s -X PUT "${BASE}/my-mysql/databases/analytics_db" \
  -H "Content-Type: application/json" -d '{}' | jq .name

# List databases
curl -s "${BASE}/my-mysql/databases" | jq '.value[].name'

# Clean up
curl -s -X DELETE "${BASE}/my-mysql"
```

## Limitations

- Stub only -- no real MySQL backend. Server and database metadata is stored in memory.
- Request body properties (charset, collation, SKU) are accepted and reflected in responses but have no backend effect.
- No firewall rules, private endpoints, or replication.
