---
name: contract-first-api-design
description: Use when planning a new API or new endpoints — define the OpenAPI spec as part of the planning phase before any implementation code, so design decisions are reviewed and agreed upon first
---

# Contract-First API Design

Define the API spec **before** writing code. The spec is the source of truth — code conforms to it.

**Not for:** Internal-only single-consumer utilities or throwaway prototypes. If another team consumes it or it goes to production, write a spec.

## Why Spec First

- Design issues are cheap to fix in YAML, expensive after services depend on it
- Frontend and backend work in parallel against the same contract
- Documenting after is 2-3x slower; a focused spec takes 1-2 hours

## Workflow

1. **Write** the OpenAPI spec
2. **Review** via PR with stakeholders and consumers
3. **Implement** — backend builds to match; frontend mocks via Prism/Stoplight
4. **Validate** — contract tests (Dredd/Schemathesis) + spec linting (Spectral) in CI

## Example

```yaml
openapi: 3.0.3
info: { title: Orders API, version: 1.0.0 }
security: [{ bearerAuth: [] }]
paths:
  /orders:
    get:
      summary: List orders
      parameters:
        - { name: status, in: query, schema: { type: string, enum: [pending, shipped, delivered] } }
        - { name: limit, in: query, schema: { type: integer, default: 25, maximum: 100 } }
      responses:
        '200':
          description: Order list
          content:
            application/json:
              schema: { $ref: '#/components/schemas/OrderList' }
              example: { items: [{ id: "order-001", status: "shipped", total: 59.99 }], total: 42 }
        '400': { $ref: '#/components/responses/Error' }
        '401': { $ref: '#/components/responses/Error' }
    post:
      summary: Create order
      requestBody:
        required: true
        content:
          application/json:
            schema: { $ref: '#/components/schemas/CreateOrderRequest' }
            example: { customerId: "cust-42", items: [{ sku: "WIDGET-1", qty: 2 }] }
      responses:
        '201':
          description: Order created
          headers:
            Location: { schema: { type: string }, example: "/orders/order-001" }
          content:
            application/json:
              schema: { $ref: '#/components/schemas/Order' }
        '400': { $ref: '#/components/responses/Error' }
        '422': { $ref: '#/components/responses/Error' }
components:
  securitySchemes:
    bearerAuth: { type: http, scheme: bearer, bearerFormat: JWT }
  responses:
    Error:
      description: Error response
      content:
        application/json:
          schema:
            type: object
            properties:
              error:
                type: object
                required: [code, message]
                properties:
                  code: { type: string, example: "VALIDATION_ERROR" }
                  message: { type: string }
                  details: { type: array, items: { type: object } }
                  correlationId: { type: string }
```

**Must include:** paths/methods, request/response schemas per status code, error format, auth, examples.

## Spec Evolution

Update spec first, then code — for existing APIs too, not just greenfield.

**Non-breaking (within current version):** add optional field, add optional param, add new endpoint.
**Breaking (new major version):** remove/rename field, change field type, add required request field.

**Deprecation:** `deprecated: true` in spec → `Sunset` header → communicate timeline → maintain → remove in next major.

## Validation

- Spec linting (Spectral) for naming and completeness
- Contract tests (Dredd/Schemathesis) to verify implementation matches spec
- CI fails if spec and implementation diverge
- Spec lives in repo, PR-reviewed alongside code
