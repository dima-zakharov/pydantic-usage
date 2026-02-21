# Pydantic v2 Usage Summary

**Project:** MCP Context Forge  
**Pydantic Version:** v2  
**Generated:** 2026-02-21

---

## Overview

This project uses Pydantic v2 extensively for data validation, serialization, and configuration management. The codebase contains over 200 Pydantic model definitions across schemas, configuration, and plugin frameworks.

---

## Base Classes

| Class | Location | Purpose |
|-------|----------|---------|
| `BaseModel` | `pydantic` | Core model inheritance |
| `BaseSettings` | `pydantic_settings` | Environment-based configuration |
| `BaseModelWithConfigDict` | `mcpgateway/utils/base_models.py` | Custom base with MCP protocol config |

---

## Configuration Classes

### ConfigDict
Used for model-level configuration:

```python
model_config = ConfigDict(
    from_attributes=True,        # ORM mode compatibility
    alias_generator=to_camel_case,  # snake_case → camelCase
    populate_by_name=True,       # Accept both naming conventions
    use_enum_values=True,        # Serialize enums as values
    extra="ignore",              # Ignore extra fields
)
```

### SettingsConfigDict
Used for settings configuration:

```python
model_config = SettingsConfigDict(
    env_file=".env",
    env_file_encoding="utf-8",
    case_sensitive=False,
    extra="ignore",
)
```

---

## Field Types & Validators

### Field Definitions
```python
from pydantic import Field, SecretStr, HttpUrl, AnyHttpUrl, EmailStr, PositiveInt

# With validation
port: PositiveInt = Field(default=4444, ge=1, le=65535)
name: str = Field(..., min_length=1, max_length=255, description="Resource name")

# Specialized types
basic_auth_password: SecretStr
database_url: HttpUrl
api_base_url: AnyHttpUrl
admin_email: EmailStr
```

### Field Validators

**Before mode** - Transform input before validation:
```python
@field_validator("name", mode="before")
@classmethod
def normalize_name(cls, v: str) -> str:
    return v.strip().lower()
```

**After mode** - Validate/transform after field validation:
```python
@field_validator("tags")
@classmethod
def validate_tags(cls, v: List[str]) -> List[str]:
    return [tag.strip() for tag in v if tag.strip()]
```

**Usage count:** ~75+ field validators across the codebase

### Model Validators

**Before mode** - Validate/transform entire model before field validation:
```python
@model_validator(mode="before")
@classmethod
def normalize_auth(cls, data: Any) -> Any:
    if isinstance(data, dict):
        data["auth_type"] = data.get("auth_type", "basic")
    return data
```

**After mode** - Cross-field validation after all fields validated:
```python
@model_validator(mode="after")
def validate_auth_credentials(self) -> Self:
    if self.auth_type == "basic" and not self.auth_value:
        raise ValueError("Basic auth requires auth_value")
    return self
```

**Usage count:** ~35+ model validators across the codebase

### Field Serializers
Custom serialization logic:
```python
@field_serializer("timestamp")
def serialize_timestamp(self, v: datetime) -> str:
    return v.isoformat()

@field_serializer("server_ids", "tenant_ids")
def serialize_sets(self, v: Set[str]) -> List[str]:
    return list(v)
```

---

## Serialization & Deserialization Methods

### Model → Data

| Method | Purpose | Common Options |
|--------|---------|----------------|
| `model_dump()` | Convert to dictionary | `by_alias`, `exclude_none`, `exclude_unset` |
| `model_dump_json()` | Convert to JSON string | `indent`, `by_alias`, `exclude_none` |

**Examples:**
```python
# Basic serialization
data = tool.model_dump()

# With aliases (camelCase for API responses)
response = tool.model_dump(by_alias=True)

# Exclude None values
clean = resource.model_dump(exclude_none=True)

# Combined options
payload = gateway.model_dump(by_alias=True, exclude_none=True)
```

### Data → Model

| Method | Purpose | Input Type |
|--------|---------|------------|
| `model_validate()` | Validate and create instance | `dict`, ORM object |
| `model_validate_json()` | Validate and create instance | JSON string |

**Examples:**
```python
# From dictionary
tool = ToolCreate.model_validate(request_data)

# From JSON string
config = Settings.model_validate_json(json_string)

# With context (for validators)
obj = MyModel.model_validate(data, context={"user": current_user})
```

---

## Key Model Files

| File | Lines | Description |
|------|-------|-------------|
| `mcpgateway/schemas.py` | 7,753 | API request/response schemas |
| `mcpgateway/config.py` | 2,408 | Application settings |
| `mcpgateway/plugins/framework/models.py` | 1,200+ | Plugin system models |
| `mcpgateway/common/models.py` | 800+ | Shared MCP protocol types |
| `mcpgateway/utils/base_models.py` | 120 | Base model utilities |

---

## Model Categories

### Metrics Schemas
- `ToolMetrics` - Tool invocation statistics
- `ResourceMetrics` - Resource access statistics
- `ServerMetrics` - Server performance metrics
- `PromptMetrics` - Prompt usage metrics
- `A2AAgentMetrics` - Agent interaction metrics

### Entity Schemas
- **Tools:** `ToolCreate`, `ToolUpdate`, `ToolRead`, `ToolInvocation`
- **Resources:** `ResourceCreate`, `ResourceUpdate`, `ResourceRead`, `ResourceContent`
- **Prompts:** `PromptCreate`, `PromptUpdate`, `PromptRead`, `PromptArgument`
- **Gateways:** `GatewayCreate`, `GatewayUpdate`, `GatewayRead`
- **Agents:** `A2AAgentCreate`, `A2AAgentUpdate`, `A2AAgentRead`

### RPC & Transport
- `RPCRequest`, `RPCResponse` - JSON-RPC message formats
- `EventMessage` - Event streaming payloads
- `CreateMessageResult` - MCP protocol responses

### Admin & Auth
- `UserCreate`, `UserRead`, `UserUpdate`
- `TeamCreate`, `TeamRead`, `TeamUpdate`
- `TokenPayload`, `JWTPayload`
- `OAuthConfig`, `SSOSettings`

---

## Common Patterns

### Snake Case ↔ Camel Case Conversion
```python
class BaseModelWithConfigDict(BaseModel):
    model_config = ConfigDict(
        alias_generator=to_camel_case,
        populate_by_name=True,
    )
```

### Secret Handling
```python
from pydantic import SecretStr

class Settings(BaseSettings):
    jwt_secret_key: SecretStr
    basic_auth_password: SecretStr
    
    # Access value
    def get_secret(self) -> str:
        return self.jwt_secret_key.get_secret_value()
```

### Validation Info Access
```python
@field_validator("auth_value", mode="before")
@classmethod
def validate_auth_value(cls, v, info: ValidationInfo) -> str:
    auth_type = info.data.get("auth_type")
    # Validate based on auth_type
    return v
```

### Self Return for Post-Validation
```python
@model_validator(mode="after")
def validate_cross_fields(self) -> Self:
    if self.field_a and not self.field_b:
        raise ValueError("field_b required when field_a is set")
    return self
```

---

## Plugin Framework Integration

The plugin system uses Pydantic for configuration and context:

```python
from mcpgateway.plugins.framework import PluginConfig, PluginContext

class PluginConfig(BaseModel):
    name: str
    enabled: bool
    config: Dict[str, Any]

class PluginContext(BaseModel):
    request_id: str
    user_id: Optional[str]
    metadata: Dict[str, Any]
```

---

## Testing Patterns

### Model Validation Testing
```python
from pydantic import ValidationError

def test_valid_tool_creation():
    tool = ToolCreate(name="test", url="https://example.com")
    assert tool.name == "test"

def test_invalid_tool_creation():
    with pytest.raises(ValidationError) as exc_info:
        ToolCreate(name="", url="invalid")
    errors = exc_info.value.errors()
    assert len(errors) > 0
```

### Settings Testing
```python
def test_settings_from_env(monkeypatch):
    monkeypatch.setenv("BASIC_AUTH_PASSWORD", "test-secret")
    settings = Settings()
    assert settings.basic_auth_password.get_secret_value() == "test-secret"
```

---

## Dependencies

```toml
[tool.poetry.dependencies]
pydantic = "^2.x"
pydantic-settings = "^2.x"
```

---

## Best Practices Observed

1. **Use `model_dump(by_alias=True)`** for API responses to maintain camelCase convention
2. **Use `SecretStr`** for all sensitive configuration values
3. **Prefer `mode="after"` validators** for cross-field validation
4. **Use `ValidationInfo`** to access other field values in field validators
5. **Return `Self`** from `@model_validator(mode="after")` methods
6. **Use `Field()`** with descriptive `description` for all schema fields
7. **Leverage `exclude_none=True`** to reduce payload size
8. **Use `from_attributes=True`** for ORM model compatibility

---

## References

- [Pydantic v2 Documentation](https://docs.pydantic.dev/latest/)
- [Pydantic Settings](https://docs.pydantic.dev/latest/concepts/pydantic_settings/)
- [Migration Guide](https://docs.pydantic.dev/latest/migration/)
