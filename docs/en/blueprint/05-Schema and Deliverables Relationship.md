## Chapter 5: Schema and Deliverables Relationship

This chapter describes the deliverable architecture of the CAP project, specifying the central role of protocol Schema definition files within the deliverables, as well as the correspondence between Schema versions and protocol versions. Understanding the hierarchical relationships and dependencies between deliverables is the foundation for protocol implementers and community contributors to participate in project collaboration.

### 5.1 Three Types of Core Deliverables

The deliverable architecture of the CAP project consists of three types of core deliverables, forming a complete delivery chain from documentation to definition to implementation:

| Deliverable Category | Description | Repository Path Example |
|---------------------|-------------|------------------------|
| **Multi-language Protocol Documents** | Including architecture blueprints and protocol technical specifications, published equivalently in 9 languages, providing human-readable descriptions of the protocol's design intent, capability scope, and technical details | `docs/{lang}/blueprint/`, `docs/{lang}/specification/` |
| **Protocol Schema Definition Files** | The authoritative definition of protocol message formats, provided in multiple machine-readable and human-readable formats, serving as the benchmark for SDK implementation and interoperability testing | `schema/{version}/` |
| **Language-specific SDK Implementations** | Concrete programming language implementations of the protocol, developed based on Schema definition files, providing out-of-the-box protocol integration capabilities for developers across different technology stacks | `sdk/{language}/` |

Hierarchical relationships between the three types of deliverables:

- **Multi-language protocol documents** are at the strategic and specification layers, defining the top-level design of "what the protocol does" and "how it does it"
- **Protocol Schema definition files** are at the definition layer, transforming message formats from the protocol specification into precise, machine-processable formal definitions
- **Language-specific SDK implementations** are at the implementation layer, transforming protocol capabilities into directly callable programming interfaces based on Schema definition files

Updates to the three types of deliverables follow a top-down propagation path: changes in protocol documents drive Schema definition updates, and Schema definition updates drive SDK implementation adaptations.

### 5.2 Role of Schema Definition Files

Schema definition files are the **authoritative definition** of CAP protocol message formats, serving the following core roles within the project deliverable architecture:

- **Single Source of Truth for Message Formats**: Schema definition files precisely describe the field names, data types, required/optional constraints, and nested structures of all message types in the CAP protocol. When ambiguity exists between the textual description in protocol documents and the Schema definition, the Schema definition takes precedence
- **Development Benchmark for SDK Implementation**: The message serialization/deserialization logic of each language SDK must strictly follow the Schema definition. SDK developers use Schema definition files to understand the precise structure of each message, ensuring that SDK implementations across different languages are completely consistent in message format
- **Verification Benchmark for Interoperability Testing**: Interoperability testing between different language SDKs uses the Schema definition as the verification standard. A message serialized by one SDK must be correctly deserialized by another SDK, and the Schema definition file is the sole basis for determining "correctness"
- **Foundation for Automated Verification**: Schema definition files (especially schema.json) can be directly consumed by automated tools for automatic message format validation, code generation, and documentation generation, reducing human errors

### 5.3 Schema Version and Protocol Version Correspondence

Each officially released CAP protocol version corresponds to a Schema version, with Schema files stored in directories named after the protocol version release date, ensuring clear and traceable version correspondence.

**Directory Naming Rules:**

| Protocol Status | Schema Directory | Description |
|----------------|-----------------|-------------|
| Draft | `schema/draft/` | Schema definitions under development, subject to change at any time, with no compatibility commitments |
| Official Release | `schema/{YYYY-MM-DD}/` | Named by release date, e.g., `schema/2025-10-25/`, immutable after release |

**Version Correspondence Rules:**

- **One-to-One Correspondence**: Each officially released protocol version corresponds to exactly one Schema version directory, with the directory name using the release date of that protocol version
- **Draft Sharing**: All Schema changes in the draft phase share the `schema/draft/` directory; the draft phase does not distinguish version numbers
- **Immutability**: Files in officially released Schema directories are not modified after release. Any Schema changes (including error corrections) are implemented through new version releases
- **Historical Retention**: Old version Schema directories are permanently retained in the repository for historical reference and backward compatibility verification

**Current Version Correspondence:**

| Protocol Version | Schema Directory | Status |
|-----------------|-----------------|--------|
| draft | `schema/draft/` | In Development |
| 2025-10-25 | `schema/2025-10-25/` | Released |

### 5.4 Schema File Format Description

Each Schema version directory contains 3 format Schema definition files, each targeting different use cases and consumers:

| Format | Filename | Purpose | Primary Consumers |
|--------|----------|---------|-------------------|
| **JSON Schema** | `schema.json` | Machine-readable message format definition for automated validation, code generation, and cross-language interoperability verification | Automated tools, CI/CD pipelines, SDK code generators |
| **TypeScript** | `schema.ts` | TypeScript type definitions providing native type support for the TypeScript/JavaScript ecosystem, used for TS/JS SDK development and IDE type checking | TypeScript/JavaScript SDK developers, frontend integration developers |
| **MDX** | `schema.mdx` | Renderable document format presenting Schema definitions in a human-friendly manner, used for documentation site display and protocol learning | Protocol learners, documentation sites, technical reviewers |

**Relationships Between the Three Formats:**

- **schema.json is the authoritative format**: When discrepancies exist between the three formats, schema.json takes precedence. schema.ts and schema.mdx should maintain semantic consistency with schema.json
- **schema.ts is the development convenience format**: schema.ts converts the type definitions in schema.json into TypeScript native types, enabling TS/JS developers to directly obtain type hints and compile-time checks in their IDEs
- **schema.mdx is the presentation format**: schema.mdx converts the structured definitions in schema.json into renderable document content, supporting interactive browsing of Schema definitions on documentation sites

**File Directory Structure Example:**

```
schema/
├── draft/
│   ├── schema.json
│   ├── schema.ts
│   └── schema.mdx
└── 2025-10-25/
    ├── schema.json
    ├── schema.ts
    └── schema.mdx
```
