# Design philosophy

The major architectural techniques and design patterns are:
- Domain driven design (DDD)
- Ports and adapters

The design pattern we **MUST NOT** use:
- Event-driven architecture

We prioritize business domain logic and avoid overengineering in system design and development.

Use simple, explicit code flows to facilitate debugging and maintenance.

Each component must only modify its internal data or inbound data, and process data using pure functions without side effects.

To orchestrate logic between components, write actions that clearly describe the flow and interactions. This makes code robust and easy to understand, avoiding magic, hidden interactions, or hard-to-track behavior.

# Core components

## Store

in-memory storage instance that:
- Store orders, positions, and more.
- Support both read and write operations with optimized access patterns.
- Trigger the processing data pipeline.

## Data

- Collect data types (quotes, bars, order books, custom data, and more) from multiple sources.
- Store the collected data types into the global store

## OrderExecution

## Risk


# Action List

## Naming Convention

The name of action follow the some templates:

- `[Vert] + [Object] + [TargetObject/Destination]`, 
    - `Verb`: The action being taken (e.g., add, calculate, convert, export).
    - `Context/Object`: What is being acted upon (e.g., Bar, Position, Report).
    - `Target/Destination`: (Optional) Where it’s going or how it’s changing (e.g., ToStore, AsPdf).
    - For example:
        - `AddNewBarToStore`

- [Get/Find/Search] + [Resource] + By + [Criteria]
    - `Get`: Usually implies a fast, direct lookup (like from a database or cache by ID).
    - `Find/Search`: Implies it might require a query, computation, or could return empty/null.
    - For example: 
        - `searchProductsByKeyword(query)`
        - `finddAtiveOrdersByCustomer(customerId)`