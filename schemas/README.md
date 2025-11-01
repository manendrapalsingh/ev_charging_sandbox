# JSON Schemas for EV Charging Network APIs

This directory contains JSON Schema definitions for the EV Charging Network domain, following the Beckn Protocol specification. These schemas define the structure, validation rules, and data types for all API requests and responses in the EV charging ecosystem.

## Directory Structure

```
schemas/
└── ev_charging_network/
    └── v1.0.0/
        ├── all.json              # Combined schema with all API definitions
        ├── discover.json         # Discover charging stations schema
        ├── select.json           # Select charging station schema
        ├── init.json             # Initialize session schema
        ├── confirm.json          # Confirm booking schema
        ├── update.json           # Update booking schema
        ├── cancel.json           # Cancel booking schema
        ├── track.json            # Track session schema
        ├── support.json          # Support request schema
        ├── rating.json           # Rating submission schema
        ├── on_discover.json      # Response to discover request
        ├── on_search.json        # Response to search request
        ├── on_select.json        # Response to select request
        ├── on_init.json          # Response to init request
        ├── on_confirm.json       # Response to confirm request
        ├── on_update.json        # Response to update request
        ├── on_cancel.json        # Response to cancel request
        ├── on_track.json         # Response to track request
        ├── on_status.json        # Status update callback schema
        ├── on_support.json       # Response to support request
        └── on_rating.json        # Response to rating request
```

## Schema Overview

### Schema Standard

All schemas in this directory follow the **JSON Schema Draft 2020-12** specification, identified by:

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema"
}
```

### Version Information

- **Domain**: EV Charging Network
- **Current Version**: v1.0.0
- **Protocol**: Beckn Protocol

## API Schema Categories

### BAP (Buyer App Provider) Schemas

These schemas define requests sent by the buyer application (user-facing app):

| Schema File | API Action | Description |
|------------|-----------|-------------|
| `discover.json` | `discover` | Search for EV charging stations based on location, route, or filters |
| `select.json` | `select` | Select a specific charging station or service |
| `init.json` | `init` | Initialize a charging session with selected parameters |
| `confirm.json` | `confirm` | Confirm and finalize the booking |
| `update.json` | `update` | Update booking details (time, duration, etc.) |
| `cancel.json` | `cancel` | Cancel an existing booking |
| `track.json` | `track` | Track the status of an active charging session |
| `support.json` | `support` | Get support information or raise support requests |
| `rating.json` | `rating` | Submit ratings and feedback for a completed session |

### BPP (Buyer Platform Provider) Schemas

These schemas define responses/callbacks sent by the seller platform (charging provider):

| Schema File | API Action | Description |
|------------|-----------|-------------|
| `on_discover.json` | `on_discover` | Response containing available charging stations |
| `on_search.json` | `on_search` | Response to search queries |
| `on_select.json` | `on_select` | Response confirming selection with details |
| `on_init.json` | `on_init` | Response with initialized session details |
| `on_confirm.json` | `on_confirm` | Response confirming the booking |
| `on_update.json` | `on_update` | Response with updated booking details |
| `on_cancel.json` | `on_cancel` | Response confirming cancellation |
| `on_track.json` | `on_track` | Response with current session status |
| `on_status.json` | `on_status` | Status update callbacks (push notifications) |
| `on_support.json` | `on_support` | Response to support queries |
| `on_rating.json` | `on_rating` | Acknowledgment of rating submission |

### Combined Schema

- **`all.json`**: A comprehensive schema file containing definitions for all API actions. Useful for complete validation or as a reference.

## Schema Structure

Each schema file follows a standard structure:

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "<action_name>",
  "type": "object",
  "properties": {
    "context": {
      "type": "object",
      "properties": {
        "version": { "type": "string" },
        "action": { 
          "type": "string",
          "enum": ["<action_name>"]
        },
        "domain": { "type": "string" },
        "location": { /* location object */ },
        "bap_id": { "type": "string" },
        "bap_uri": { "type": "string" },
        "bpp_id": { "type": "string" },
        "bpp_uri": { "type": "string" },
        "transaction_id": { "type": "string" },
        "message_id": { "type": "string" },
        "timestamp": { "type": "string" },
        "ttl": { "type": "string" },
        "schema_context": { 
          "type": "array",
          "items": { "type": "string" }
        }
      },
      "required": [ /* required context fields */ ]
    },
    "message": {
      "type": "object",
      "properties": {
        /* Action-specific message properties */
      }
    }
  },
  "required": ["context", "message"]
}
```

### Common Components

#### Context Object
Every API request/response includes a `context` object containing:
- **version**: Beckn protocol version
- **action**: The API action being performed
- **domain**: Domain identifier (e.g., "EV_CHARGING")
- **location**: Geographic location (country and city codes)
- **bap_id** / **bap_uri**: Buyer App Provider identification and endpoint
- **bpp_id** / **bpp_uri**: Buyer Platform Provider identification and endpoint
- **transaction_id**: Unique transaction identifier
- **message_id**: Unique message identifier
- **timestamp**: ISO 8601 timestamp
- **ttl**: Time-to-live for the message
- **schema_context**: Array of schema context URIs

#### Message Object
The `message` object contains action-specific data:
- For `discover`: Search filters, spatial queries, etc.
- For `on_discover`: Catalog of available charging stations
- For `init`: Session initialization parameters
- For `on_init`: Initialized session details
- And so on for other actions...

## Usage

### Validation

These schemas can be used with any JSON Schema validator to validate API requests and responses. Popular validation libraries include:

#### JavaScript/TypeScript
```javascript
// Using ajv (Another JSON Schema Validator)
import Ajv from 'ajv';
import discoverSchema from './schemas/ev_charging_network/v1.0.0/discover.json';

const ajv = new Ajv();
const validate = ajv.compile(discoverSchema);

const isValid = validate(requestPayload);
if (!isValid) {
  console.error(validate.errors);
}
```

#### Python
```python
# Using jsonschema
from jsonschema import validate
import json

with open('schemas/ev_charging_network/v1.0.0/discover.json') as f:
    schema = json.load(f)

try:
    validate(instance=request_payload, schema=schema)
    print("Valid!")
except ValidationError as e:
    print(f"Validation error: {e.message}")
```

#### Java
```java
// Using everit-org/json-schema
import org.everit.json.schema.Schema;
import org.everit.json.schema.loader.SchemaLoader;
import org.json.JSONObject;

Schema schema = SchemaLoader.load(new JSONObject(schemaJson));
schema.validate(new JSONObject(requestPayload));
```

### IDE Support

Many IDEs and editors support JSON Schema for autocomplete and validation:

- **VS Code**: Install the "JSON Schema Validator" extension
- **IntelliJ IDEA**: Built-in JSON Schema support
- **Postman**: Can import and use schemas for validation

### Integration with Postman Collections

These schemas complement the Postman collections in `/api-collection/postman-collection/`. You can:
1. Use the schemas to validate request/response payloads
2. Generate mock data from schemas
3. Set up automated schema validation in Postman tests

### Code Generation

JSON Schemas can be used to generate code in various languages:
- **TypeScript**: Using tools like `json-schema-to-typescript`
- **Java**: Using tools like `jsonschema2pojo`
- **Python**: Using tools like `datamodel-code-generator`

Example with TypeScript:
```bash
npm install -g json-schema-to-typescript
json2ts schemas/ev_charging_network/v1.0.0/discover.json -o types/discover.d.ts
```

## Schema Validation Best Practices

1. **Always validate on both ends**: Validate requests at the BPP and responses at the BAP
2. **Use strict validation**: Enable additional properties validation where possible
3. **Version management**: Keep schemas versioned and maintain backward compatibility
4. **Error handling**: Provide clear, actionable error messages from validation failures
5. **Performance**: Cache compiled schemas in production environments

## Request/Response Flow Example

Here's how schemas are used in a typical flow:

1. **BAP sends discover request**
   - Validates against `discover.json` before sending
   - BPP validates against `discover.json` on receipt

2. **BPP responds with on_discover**
   - Validates response against `on_discover.json` before sending
   - BAP validates against `on_discover.json` on receipt

3. **Continue for each action in the flow**
   - Each request validated against corresponding schema
   - Each response validated against corresponding `on_*` schema

## Schema Maintenance

### Updating Schemas

When updating schemas:
1. **Version bump**: Create a new version directory (e.g., `v1.1.0/`)
2. **Document changes**: Maintain changelog for breaking changes
3. **Test compatibility**: Ensure backward compatibility or document migration path
4. **Update documentation**: Keep this README and related docs updated

### Schema Context References

The schemas reference external schema contexts defined in the Beckn Protocol specifications:
```
https://raw.githubusercontent.com/beckn/protocol-specifications-new/refs/heads/main/schemas/charging_service/v1/context.jsonld
```

## Related Resources

- **Beckn Protocol Specification**: [https://github.com/beckn/protocol-specifications](https://github.com/beckn/protocol-specifications)
- **JSON Schema Specification**: [https://json-schema.org/](https://json-schema.org/)
- **Postman Collections**: `/api-collection/postman-collection/`
- **Swagger Documentation**: `/api-collection/swagger/`

## Tools and Libraries

### Recommended JSON Schema Validators

- **ajv** (JavaScript/TypeScript): [https://github.com/ajv-validator/ajv](https://github.com/ajv-validator/ajv)
- **jsonschema** (Python): [https://github.com/python-jsonschema/jsonschema](https://github.com/python-jsonschema/jsonschema)
- **everit-org/json-schema** (Java): [https://github.com/everit-org/json-schema](https://github.com/everit-org/json-schema)
- **gojsonschema** (Go): [https://github.com/xeipuuv/gojsonschema](https://github.com/xeipuuv/gojsonschema)

### Schema Tools

- **JSON Schema to TypeScript**: [https://github.com/bcherny/json-schema-to-typescript](https://github.com/bcherny/json-schema-to-typescript)
- **JSON Editor**: [https://json-editor.github.io/json-editor/](https://json-editor.github.io/json-editor/)
- **JSON Schema Lint**: [https://www.jsonschemavalidator.net/](https://www.jsonschemavalidator.net/)

## Notes

- All schemas use JSON Schema Draft 2020-12
- Field descriptions are provided for better understanding
- Required fields are explicitly marked in each schema
- Enum values restrict actions to valid operations
- Location codes follow ISO 3166-1 standards for countries and city codes

## Support

For questions or issues related to these schemas:
1. Refer to the Beckn Protocol documentation
2. Check the main project README
3. Review the Postman collections for example payloads
4. Contact the development team for schema-specific queries

