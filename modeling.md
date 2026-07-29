---
name: modeling
description: Design TypeDB schemas well. Use when creating or reviewing a schema - covers naming conventions, choosing entities vs relations vs attributes, subtyping vs composition, type safety strictness, role subtyping, cardinality, constraints, and functions.
---
# TypeDB Modeling Guide

## Naming of types
- TypeQL reads most elegantly when every type (entity, relation, role, and attribute) is named with a noun
- Attributes: querying `match ... has ...`, in English the object of the sentence is after the `has`, so it reads best as a noun
- Relations: querying `match ... links ...`, the object of the sentence is after the `links`, so it reads best as a noun
  - Relations read best as active or verbal nouns: for example, 'employment' is the noun form of 'to employ'; 'marriage' is the noun form of 'to marry'; sometimes these are the same, e.g. 'grant' is the noun form of 'to grant'; 'review' is the noun of 'to review' the same way; 'assignment' is the noun form of 'to assign'
- In other words, the main TypeQL keywords are verbs, so the types slot in as nouns around them
- Functions should be named by what they return and the constraints they contain, e.g. `persons_by_name`, or `principals_with_access`

## When to use Entities vs Relations vs Attributes
- TypeDB treats relations and attributes as first-class, alongside entities
- Use entities when the piece of data has an independent existence, unrelated to any relationships or specific attributes
- Use relations when the existence of the relationship depends on the linked role players. TypeDB auto-deletes any relations with 0 role players, even if they have attributes.
  - TypeDB relations are n-ary, meaning you never need to reify relationships.
  - You can nest relations, i.e. make relations role players in other relations.
- Attributes are pieces of data entirely defined by their type and their value. They are _immutable, global, shared, and linked against_. A single `name "Alice"` exists in the entire database - unrelated `person`, `dog`, and `cat` will all have links to the same attribute instance.
  - If you want to polymorphically query ownerships, then the 'global' attribute is a good model for things to share. Otherwise, you can use a specific attribute with 1 owner
  - TypeDB attributes default to dependent, and are automatically deleted when 0 owners exist of an instance. Marking the attribute type as `@independent` allows attributes to exist freely and independently - useful for example if pre-loading a dictionary of words or numbers.
- Attributes carry a value of a value type assigned in the schema, but if abstract may not need to be assigned a value type yet. A classic example is an `attribute id @abstract;`, which is later subtyped into `attribute email_id, sub id, value string;` and `attribute employee_id, sub id, value integer;`

## Subtyping vs Composition
- TypeDB supports only single-inheritance. Treat this as a true, independent "is-a" axis.
- Consider relations' roles that are played as a kind of _trait_ or _interface_. That means, to compose, we add linked relations and play roles. This can be viewed as implementing a component architecture in OOP.
  - For example, we might have `entity person`, but a person can behave as an employee: `entity person, plays employment:employee`.
  - Similarly, we can consider attribute ownerships a type of interface: `entity person, owns name` indicates a person can be a "name-owner"
  - Both approaches bring about read-time polymorphism: `$x isa person;` will resolve subtype polymorphism; `$x has name $n` or `$r links (employee: $x)` will resolve interface polymorphism
- A nice way to implement 'multi-typing' in TypeQL is to create unary relation types that act as components for a type, e.g. `entity person, plays hr_component:employee @card(0..1)`, where a strongly-typed system can be built off the person's `HR` representation (of which at most 1 can exist)

## Type safety strictness
- TypeDB gives you the power to choose the level of type safety you want
- You can choose to avoid defining specialized types and create an entity type `thing` with an attribute `label` - you trade off compiler errors for silent misses and data bugs.
- You can choose to enumerate types into the schema, and the database will type check those for you
- You can even turn attributes with enumerated values into unary relation types and the system will validate the enums as types instead

The level of type safety you require for your application is defined by the size of the schema you want to manage and the level of rigor you want to have your data reads and writes validated at. More types buy you earlier errors and richer queries; less gets you flexibility.

## Role subtyping
- Role types are defined within a "scope" of a relation type, but are thereafter "true" types. I.e. `relation employment relates employee` creates a role type `employment:employee` that can be queried
- Role types are inherited 'as-is', i.e. a `part_time_employment sub employment` inherits the role `employment:employee`. A `full_time_employment` also does. The role `part_time_employment:employee` is not a real role - it is actually the same `employment:employee` held by `employment`, `part_time_employment`, and `full_time_employment`
  - To create a specialized role just for the sub-relation type, we create a role subtype: `part_time_employment relates part_time_employee as employee`, which essentially creates a new role type `part_time_employment:part_time_employee sub employment:employee`, and also blocks access to the inherited `employee` role in `part_time_employment` instances. However, because of the subtyping, read-time `match $r links (employee: $x)` will resolve also to players of `part_time_employment:part_time_employee`! The cost is a new name must be created for the sub-role type

## Cardinality
- Use the most specific possible cardinality for owns (default 0..1), relates (default 0..1), and plays (default 0..)
- Many relations will likely want to change the default @card(0..1) to @card(1) on every `relates` role type unless they are designed to be optional roles
- The only way to encode a 'required' connection is with `@card(1)` or higher counts eg. `@card(1..)`  (note: `@key` implies cardinality 1 as well)

## Constraints
- It can be useful to define local @values or @range or other value restrictions at the `owns` level instead of the `attribute` definition level
- Subtypes can add their own specializations of inherited capabilities, and both will be validated at commit time. For example, if an `identity` is required to have at least one `email` - `identity owns email @card(1..);`, a subtype `service_account` can set `service_account, sub identity, owns email @card(1);`. All identities are validated against locally defined and inherited constraints on the capability
- `@key` is a composition under the hood of `@unique` and `@card(1)`. `@unique` requires no other instance of this type owns the specific attribute. Key and unique go on the ownership, not the attribute.

## Functions
Functions can be used to
1) modularize huge queries
2) perform sub queries and aggregates
3) recursively query (with built-in loop detection and termination)
4) categorize/create "conceptual relations". For example, if we make a `person_permitted(person, resource, permission)` function that returns a boolean, we can use `match $x isa person ...; $r isa resource ...; $p isa permission ...; true == person_permitted($x, $r, $p);` as a sort of relation.
