---
name: modeling
description: Design TypeDB schemas well. Use when creating or reviewing a schema - covers naming conventions for types and functions, choosing the right level of type safety, generic vs specific attributes, cardinality defaults, and where to place constraints.
---
# TypeDB Modeling Guide

## Naming of types
- TypeQL reads most elegantly when every type (entity, relation, role, and attribute) is named with a noun
- Attributes: querying `match ... has ...`, in English the object of the sentence is after the `has`, so it reads best as a noun
- Relations: querying `match ... links ...`, the object of the sentence is after the `links`, so it reads best as a noun
  - Relations read best as active or verbal nouns: for example, 'employment' is the noun form of 'to employ'; 'marriage' is the noun form of 'to marry'; sometimes these are the same, e.g. 'grant' is the noun form of 'to grant'; 'review' is the noun of 'to review' the same way; 'assignment' is the noun form of 'to assign'
- In other words, the main TypeQL keywords are verbs, so the types slot in as nouns around them
- Functions should be named by what they return and the constraints they contain, e.g. `persons_by_name`, or `principals_with_access`

## Type safety strictness
- TypeDB gives you the power to choose the level of type safety you want
- You can choose to avoid defining specialized types and create an entity type `thing` with an attribute `label` - you trade off compiler errors for runtime errors
- You can choose to enumerate types into the schema, and the database will type check those for you
- You can even turn attributes with enumerated values into unary relation types and the system will validate the enums as types instead

The level of type safety you require for your application is defined by the size of the schema you want to manage and the level of rigor you want your data reads and writes to be validated at. More types buy you earlier errors and richer queries; less gets you flexibility.

## Generic and shared vs specific attributes
- When do you use a general `start-date` and share it across different owners, versus a `policy-start-date`, which is specific to one owner type?
  - Future: scoped attributes would be nice, entity `policy` --> attribute `policy.start-date`
- We struggled to find use cases where attributes were truly 'global':
  - If an attribute is a "first-class" citizen, such as `company-id`, this exposes an interface "company-id ownership" that multiple things might implement
  - If an attribute is purely an 'embedded property', then it's kind of a "scoped attribute"
- Modeling guide: if you want to polymorphically query ownerships, then the 'global' attribute is a good model for things to share. Otherwise, you can use a specific attribute with 1 owner (or in the future, the idea of a 'scoped' attribute)

## Cardinality
- Use the most specific possible cardinality for owns (default `0..1`), relates (default `0..1`), and plays (default `0..`)
- Many relations will likely want to change the default `@card(0..1)` to `@card(1)` on every `relates` role type unless they are designed to be optional roles
- The only way to encode a 'required' connection is with `@card(1)` or higher counts, e.g. `@card(1..)` (note: `@key` implies cardinality 1 as well)

## Constraints
- It can be useful to define local `@values` or `@range` or other value restrictions at the `owns` level instead of the `attribute` definition level
- Subtypes can add their own specializations of inherited capabilities, and both will be validated at commit time. For example, if an `identity` is required to have at least one `email` - `identity owns email @card(1..);` - a subtype `service-account` can set `service-account sub identity, owns email @card(1);`. All identities are validated against locally defined and inherited constraints on the capability
