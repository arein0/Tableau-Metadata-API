# Tableau Metadata API Tools

Python utilities for querying the Tableau Metadata API using cursor-based pagination. Built for use in Alteryx Python tools but works standalone as well.

---

## The Problem

The Tableau Metadata API enforces a node limit that throws an error if your query returns too many results at once.

The naive fix is hardcoding page numbers and hoping it's enough. The real fix is cursor-based pagination — let the API tell you when it's done rather than guessing. Each response tells you *"there's more, here's where you left off"* and you keep fetching until it says stop.

---

## Contents

| File | Description |
|---|---|
| `calculated_fields.py` | Pulls all calculated fields, formulas, data types, and downstream workbooks |
| `custom_sql.py` | Pulls all custom SQL datasources, parses out database/schema/table references, and maps them to workbooks |
| `embedded_datasources.py` | Pulls all embedded datasources and their upstream tables |

---

## How It Works

All scripts share the same core pagination function:

```python
def fetch_all_nodes(metadata_url, auth_token, query, connection_key, batch_size=100):
    all_nodes = []
    cursor = None

    headers = {
        "Content-Type": "application/json",
        "x-tableau-auth": auth_token,
    }

    while True:
        variables = {"first": batch_size, "after": cursor}
        response = requests.post(
            metadata_url,
            json={"query": query, "variables": variables},
            headers=headers,
            verify=False
        )
        data = response.json()
        if "errors" in data:
            raise Exception(f"GraphQL errors: {data['errors']}")
        connection = data["data"][connection_key]
        all_nodes.extend(connection["nodes"])
        if not connection["pageInfo"]["hasNextPage"]:
            break
        cursor = connection["pageInfo"]["endCursor"]

    return all_nodes
```

Each script then just defines its own GraphQL query and flattening logic — the pagination is handled automatically regardless of how many nodes are in your Tableau environment.

---

## Requirements

- Python 3.x
- `requests`
- `pandas`
- Tableau Server with Metadata API enabled
- A valid Tableau auth token

> Authentication is handled separately. Each script expects a `metadata_url` and `auth_token` to be passed in — in Alteryx these come from a sign-in macro connected to anchor #1 of the Python tool.

---

## Usage

### Standalone Python
```python
nodes = fetch_all_nodes(
    metadata_url="https://your-tableau-server/api/metadata/graphql",
    auth_token="your-auth-token",
    query=QUERY,
    connection_key="calculatedFieldsConnection"
)
```

### In Alteryx
Connect your sign-in macro to anchor #1 of the Python tool:
```python
from ayx import Alteryx

auth_data = Alteryx.read(1)
METADATA_API_URL = auth_data["metadata_url"].iloc[0]
AUTH_TOKEN = auth_data["auth_guid"].iloc[0]
```

---

## Output Structure

### calculated_fields.py
| Column | Description |
|---|---|
| `field_id` | Unique identifier for the calculated field |
| `field_name` | Name of the calculated field |
| `description` | Field description |
| `data_type` | Data type (string, integer, etc.) |
| `formula` | The calculation formula |
| `datasource_id` | ID of the parent datasource |
| `datasource_name` | Name of the parent datasource |
| `workbook_id` | ID of the downstream workbook |
| `workbook_name` | Name of the downstream workbook |
| `workbook_project` | Project the workbook belongs to |
| `workbook_uri` | Workbook URI on Tableau Server |

### custom_sql.py
**Output 1 — Parsed table references:**
| Column | Description |
|---|---|
| `workbook_name` | Workbook containing the custom SQL |
| `datasource_name` | Datasource the custom SQL belongs to |
| `custom_sql_name` | Name of the custom SQL table |
| `server` | Server derived from upstream database connection |
| `database` | Database parsed from SQL table reference |
| `schema` | Schema parsed from SQL table reference |
| `table` | Table name parsed from SQL table reference |
| `raw_ref` | Raw table reference as captured from SQL |

**Output 2 — Query level:**
| Column | Description |
|---|---|
| `workbook_name` | Workbook containing the custom SQL |
| `datasource_name` | Datasource the custom SQL belongs to |
| `custom_sql_name` | Name of the custom SQL table |
| `query` | The full custom SQL query text |

---

## Notes

- **SSL verification** is disabled by default (`verify=False`) for compatibility with internal Tableau Servers using self-signed certificates. Replace with your CA bundle path in production:
  ```python
  verify=r"C:\path\to\your\company-ca-bundle.crt"
  ```
- **Server** is derived from Tableau's upstream database connection metadata rather than parsed from SQL, since server is rarely written into SQL directly
- **Null datasources** are handled gracefully — some calculated fields (ad-hoc, worksheet-level, or parameters) have no datasource and will return null for datasource fields
- **Batch size** defaults to 100 but can be tuned down if you hit node limits on deeply nested queries

---

## Related Resources

- [Tableau Metadata API Documentation](https://help.tableau.com/current/api/metadata_api/en-us/index.html)
- [Tableau GraphQL Schema Reference](https://help.tableau.com/current/api/metadata_api/en-us/reference/index.html)
