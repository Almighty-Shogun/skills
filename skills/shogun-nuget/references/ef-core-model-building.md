# AlmightyShogun.EntityFrameworkCore.ModelBuilding

`ModelBuilder` extension methods for the relationship, index and enum
configuration that otherwise makes `OnModelCreating` noisy. One file, no state,
no registration: every helper returns the `ModelBuilder` so calls chain.

Docs: https://nuget.docs.shogun.ms/ef-core-model-building/

Depends only on EF Core and EF Core relational. It references no other package in
this scope.

## What is where

| Need | Reach for |
| --- | --- |
| One to one | `ApplyOneToOne<TEntity, TDependent>(navigation, foreignKey, inverse?)` |
| One to many | `ApplyOneToMany<TEntity, TDependent>(navigation, foreignKey, inverse?)` |
| Many to one, configured from the dependent | `ApplyManyToOne<TEntity, TDependent>(navigation, foreignKey, inverse?)` |
| Many to many with an explicit join table | `ApplyManyToMany<TEntity, TRelated>(navigation, inverse, joinTableName)` |
| Always load a navigation | `ApplyAutoInclude<TEntity>(navigation)` |
| Index | `ApplyIndex<TEntity>(index)` |
| Unique index, optionally filtered | `ApplyUniqueIndex<TEntity>(index, filter?)` |
| Store an enum as text | `ApplyEnumAsString<TEntity, TProperty>(property, maxLength = 32)` |

## Usage

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    base.OnModelCreating(modelBuilder);

    modelBuilder
        .ApplyOneToMany<Order, OrderLine>(order => order.Lines, line => line.OrderId, line => line.Order)
        .ApplyManyToOne<Customer, Order>(order => order.Customer, order => order.CustomerId)
        .ApplyUniqueIndex<Customer>(customer => customer.Email)
        .ApplyUniqueIndex<Customer>(customer => customer.VatNumber, "\"VatNumber\" IS NOT NULL")
        .ApplyEnumAsString<Order, OrderStatus>(order => order.Status)
        .ApplyAutoInclude<Order>(order => order.Customer);
}
```

Each relationship helper has a second overload taking a principal key expression
as a fourth argument, for a relationship that does not point at the primary key.
On that overload the inverse navigation is a positional parameter, so pass `null`
explicitly when there is none.

## Traps

**`ApplyOneToMany` and `ApplyManyToOne` differ only in which side you name
first.** `ApplyOneToMany` is written from the principal (`Order` has many
`OrderLine`), `ApplyManyToOne` from the dependent (`Order` has one `Customer`).
Both configure the same relationship; pick the one whose navigation you have.

**`ApplyManyToMany` names the join table and its columns for you.** Foreign keys
are `<TRelated>Id` and `<TEntity>Id` from the type names, so renaming an entity
renames a column and needs a migration.

**A composite index or key is an anonymous object**: `entity => new { entity.A,
entity.B }`. The expression is typed `object?`, so this compiles and behaves the
way EF Core expects.

**`ApplyUniqueIndex`'s `filter` is raw provider SQL**, passed straight to
`HasFilter`, so it is provider specific and not validated here.

**`ApplyEnumAsString` sets `HasMaxLength(32)` by default.** An enum with longer
member names needs the explicit length or the column truncates.

**These helpers do not set delete behavior, required or optional.** They call
`HasOne`, `WithMany`, `HasForeignKey` and nothing else, so EF Core conventions
decide the rest. Chain the ordinary fluent API when you need more.

**Nothing here registers anything.** It is a static extension class; there is no
service and no assembly scan.

## Public surface

Extensions on `ModelBuilder`: `ApplyOneToOne`, `ApplyOneToMany`,
`ApplyManyToOne` (each with a principal-key overload), `ApplyManyToMany`,
`ApplyAutoInclude`, `ApplyIndex`, `ApplyUniqueIndex`, `ApplyEnumAsString`.
