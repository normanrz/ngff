# JSON-LD in RFC-8: Summary

This document summarizes the adoption of JSON-LD in RFC-8, including benefits, changes made, and potential concerns.

## Benefits of Using JSON-LD

### Standardized Type System
- `@type` provides a well-defined mechanism for declaring object types
- Types are mapped to IRIs via contexts, ensuring global uniqueness
- Standard tooling exists for validating and processing JSON-LD documents

### Standardized Identifiers and References
- `@id` provides a consistent way to define and reference objects
- Identifiers can be simple strings or full IRIs
- Context can define properties as references (`"@type": "@id"`), allowing simple string syntax while maintaining proper semantics
- JSON-LD processors understand which properties are references vs. literals

### Extensibility via Contexts
- Extensions are managed through JSON-LD contexts (`@context`)
- Custom vocabularies can be published at URLs and reused across documents
- Scoped contexts allow the same term to have different meanings in different locations (e.g., `zarr` as a path type vs. node type)
- No need for a custom prefix registry - just publish a context file

### Namespace Management
- Prefixes (e.g., `mobie:grid`) clearly distinguish custom terms from core terms
- Prefixes expand to full IRIs, preventing naming collisions
- Multiple organizations can extend the spec without coordination

### Separation of Concerns
- `@context` inside `ome` object keeps JSON-LD scoped to OME-NGFF metadata
- Zarr-specific fields (`zarr_format`, `node_type`) remain untouched
- Clean separation in `zarr.json` files

### Interoperability
- JSON-LD is a W3C standard with broad tooling support
- Documents can be processed by generic JSON-LD processors
- Potential for integration with semantic web tools and linked data ecosystems

## Changes Made to RFC-8

### Metadata Fields
| Before | After |
|--------|-------|
| `"type": "collection"` | `"@type": "Collection"` |
| `"type": "multiscale"` | `"@type": "Multiscale"` |
| `"type": "singlescale"` | `"@type": "Singlescale"` |
| `"id": "myid"` | `"@id": "myid"` |

### Path Objects
| Before | After |
|--------|-------|
| `"type": "zarr"` | `"@type": "zarr"` |
| `"type": "json"` | `"@type": "json"` |

### Coordinate Systems and Transformations
| Before | After |
|--------|-------|
| `"id": "world"` | `"@id": "world"` |
| `"type": "translation"` | `"@type": "translation"` |

### Internal References
Reference properties like `input`, `output`, and `source` are defined in the context with `"@type": "@id"`:
```jsonc
// In the context file:
{
    "@context": {
        "input": {"@type": "@id"},
        "output": {"@type": "@id"},
        "source": {"@type": "@id"}
    }
}
```

This means JSON-LD processors understand that string values are references to `@id` values, not literals.

### Document Structure
- Added `@context` inside the `ome` object (required)
- Context references `https://ngff.openmicroscopy.org/0.x/context.jsonld`
- Custom extensions add additional contexts to the array

### Extensibility
- JSON-LD contexts replace the custom prefix registry concept
- Custom terms SHOULD always use prefixes (e.g., `mobie:grid` instead of `grid`)
- Extensions define a prefix for their namespace in the context array

### Example: Before
```jsonc
{
    "ome": {
        "version": "0.5",
        "type": "collection",
        "name": "example",
        "nodes": [{
            "type": "multiscale",
            "id": "image1",
            "path": {
                "type": "zarr",
                "path": "./image1"
            }
        }]
    }
}
```

### Example: After
```jsonc
{
    "ome": {
        "@context": "https://ngff.openmicroscopy.org/0.x/context.jsonld",
        "version": "0.x",
        "@type": "Collection",
        "name": "example",
        "nodes": [{
            "@type": "Multiscale",
            "@id": "image1",
            "path": {
                "@type": "zarr",
                "path": "./image1"
            }
        }]
    }
}
```

### Example: Custom Extensions
```jsonc
{
    "ome": {
        "@context": [
            "https://ngff.openmicroscopy.org/0.x/context.jsonld",
            { "mobie": "https://mobie.github.io/vocab#" }
        ],
        "@type": "Collection",
        "attributes": {
            "mobie:grid": true,           // Custom attribute
            "mobie:voxelType": "labels"   // Custom attribute
        },
        "nodes": [{
            "@type": "mobie:Table",       // Custom node type
            "name": "measurements",
            "path": {
                "@type": "mobie:parquet", // Custom path type
                "path": "./measurements.parquet"
            }
        }]
    }
}
```

## Hybrid Approach: What's JSON-LD vs. OME-NGFF-Specific

### Standard JSON-LD
- `@context` - vocabulary definition
- `@type` - type declarations
- `@id` - identifiers
- Prefix expansion (e.g., `mobie:grid` → `https://mobie.github.io/vocab#grid`)
- Scoped contexts for differentiating types in different locations
- Reference properties defined with `"@type": "@id"` (e.g., `input`, `output`, `source`)

### OME-NGFF Extensions (Not Standard JSON-LD)
- **Path objects**: The `path` mechanism with `@type` (zarr, json) and `path` field
- **External references**: Combining `@id` with a `Path` object
- **Path resolution**: Relative paths, local files, appending `zarr.json`

Standard JSON-LD uses IRIs for linking, which doesn't support our requirements for relative paths and different storage formats.

## Potential Downsides and Concerns

### Verbosity
- Every document needs `@context`
- `@type` and `@id` are slightly longer than `type` and `id`
- Type values are now PascalCase (`Collection` vs `collection`)

### Learning Curve
- JSON-LD concepts may be unfamiliar to some users
- Understanding contexts, prefixes, and scoped contexts requires learning
- Documentation needs to explain both JSON-LD basics and OME-NGFF specifics

### Hybrid Complexity
- The `Path` mechanism is NOT standard JSON-LD, which may confuse users
- Users might expect standard JSON-LD linking to work
- Need to clearly document what's JSON-LD vs. OME-NGFF-specific

### Context Dependency
- Documents depend on remote context files being available
- Context URL changes could break documents (though versioned URLs mitigate this)
- Offline processing requires caching or embedding contexts

### Tooling Expectations
- Generic JSON-LD processors won't understand our custom `Path` mechanism
- Full JSON-LD validation may flag our extensions as issues
- Need OME-NGFF-specific tooling for complete processing

### Inconsistencies

#### `@type` Usage
- Node types: PascalCase (`Collection`, `Multiscale`, `Singlescale`)
- Path types: lowercase (`zarr`, `json`)
- Transform types: lowercase (`scale`, `translation`)

This inconsistency exists because path and transform types feel more like "format identifiers" than "class names", but it may cause confusion.

#### Path Object `@type`
The `@type` in path objects uses JSON-LD syntax but isn't really a JSON-LD type - it's our custom path type system. This could confuse users who expect JSON-LD semantics.

#### External References
- Internal references: string with context-defined `"@type": "@id"` (standard JSON-LD)
- External references: object with `@id` and `path` (OME-NGFF extension)

This asymmetry exists because standard JSON-LD linking (IRIs) doesn't support our requirements for relative paths and different storage formats. Internal references use proper JSON-LD semantics via context definitions.

### Migration
- Existing RFC-8 implementations would need updates
- Need to handle both old (`type`, `id`) and new (`@type`, `@id`) during transition
- Version field helps distinguish, but adds complexity

## Recommendations

1. **Clear documentation**: Explicitly document what's JSON-LD vs. OME-NGFF-specific
2. **Versioned context URLs**: Use versioned URLs (e.g., `/0.x/context.jsonld`) to prevent breakage
3. **Provide context file**: Publish the actual context file, not just documentation
4. **Tooling**: Provide OME-NGFF-specific validators that understand the hybrid approach
5. **Examples**: Include many examples showing common patterns
6. **Migration guide**: Document how to migrate from non-JSON-LD RFC-8 drafts

## Open Questions

1. Should path types use PascalCase for consistency (`Zarr`, `Json`)?
2. Should the context be embeddable for offline use?
3. How should implementations handle missing/unreachable context files?
4. Should we provide JSON Schema that's aware of the JSON-LD structure?
