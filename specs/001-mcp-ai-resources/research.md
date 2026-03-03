# Research: MCP AI Resources

## R1: Resource provider return type — ReadResourceResult vs plain data

**Decision**: Use `ReadResourceResult` with `structuredContent: false`

**Rationale**: Tested simplified pattern (return self with `content`/`uri` properties). API Platform's `StructuredContentProcessor` wraps non-`ReadResourceResult` responses in `CallToolResult` — wrong for resources. The official docs show a `content`/`uri` pattern but the processor doesn't convert it to `ReadResourceResult`. Bug in API Platform pre-release. `ReadResourceResult` + `structuredContent: false` is the proven pattern.

**Alternatives tested and rejected**:
- Return self with content/uri properties (StructuredContentProcessor wraps in CallToolResult, not ReadResourceResult)
- Return plain string (same issue)

## R2: All markdown with embedded dynamic enums

**Decision**: Use `text/markdown` for all resources. Embed enum values as structured lists within markdown, generated dynamically from PHP enums at runtime.

**Rationale**:
- Markdown is natural for AI comprehension — no parsing needed
- Enum values rendered as lists within their domain context (e.g., PO statuses appear in the PO resource, not a separate enums resource)
- Dynamic generation from `::cases()` ensures zero drift
- One format simplifies the pattern — every resource works the same way

**Alternatives considered**:
- Separate JSON resources for enums (requires cross-referencing, AI reads 2 resources instead of 1)
- Mixed JSON + Markdown (two patterns to maintain, inconsistent)

## R3: Content storage — inline strings vs external files

**Decision**: Store content as string constants or heredoc in PHP provider methods

**Rationale**:
- Resources are static and don't change at runtime
- Keeping content in the provider class makes each resource self-contained
- No need for file I/O, template engines, or asset management
- Markdown content is written once and updated with code changes
- PHP heredoc syntax makes multiline markdown clean

**Alternatives considered**:
- External .md files loaded at runtime (adds file I/O, deployment concerns)
- Twig templates (unnecessary complexity for static content)
- YAML/config files (not natural for markdown content)

## R4: Dynamic enum extraction pattern

**Decision**: Use `EnumClass::cases()` with `->value` and `->name` at runtime

**Rationale**:
- PHP 8.1 backed enums provide `::cases()` which returns all values
- Many enums have additional methods (e.g., `ClinicEnum::name()`, `ProductTypeEnum::prefix()`)
- Extracting at runtime guarantees zero drift (FR-009)
- Pattern already proven in WorkflowStatuses

**Alternatives considered**:
- Hardcoded arrays (would drift from enum changes)
- Reflection-based extraction (unnecessary when `::cases()` exists)

## R5: Resource count — 8 resources by business domain

**Decision**: 8 resources organized by business domain, each self-contained

**Rationale**:
- SC-003 requires fewer than 10 resources — 8 fits
- Domain-grouped (PR, PO, receipts, returns, org, products, validation, lifecycle) matches how users think about the system
- Each resource is self-contained: AI reading "purchase-orders" gets statuses, transitions, rules, fields — no cross-referencing
- Aligns with future domain-grouped tools (~25-30 tools organized the same way)

**Alternatives considered**:
- By content type (statuses resource, entities resource, rules resource) — requires cross-referencing 3-4 resources per question
- One resource per entity (~15+ resources, too many reads)
- Fewer mega-resources (too large, mix unrelated domains)
