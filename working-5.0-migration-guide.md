Working list of changes for 5.0
===================================

* Switch from Configuration to ServiceRegistry+Metadata for SessionFactory building
* `org.hibernate.hql.spi.MultiTableBulkIdStrategy#prepare` contract has been changed to account for Metadata
* (proposed) `org.hibernate.persister.spi.PersisterFactory` contract, specifically building CollectionPersisters)
  has been changed to account for Metadata
* extract `org.hibernate.engine.jdbc.env.spi.JdbcEnvironment` from `JdbcServices`; create
  `org.hibernate.engine.jdbc.env` package and moved a few contracts there.
* Introduction of `org.hibernate.boot.model.relational.ExportableProducer` which will effect any
 `org.hibernate.id.PersistentIdentifierGenerator` implementations
* Change to signature of `org.hibernate.id.Configurable` to accept `JdbcEnvironment` rather than just `Dialect`
* Removed deprecated `org.hibernate.id.TableGenerator` id-generator
* Removed deprecated `org.hibernate.id.TableHiLoGenerator` (hilo) id-generator
* Deprecated `org.hibernate.id.SequenceGenerator` and its subclasses
* cfg.xml files are again fully parsed and integrated (events, security, etc)
* Removed the deprecated `org.hibernate.cfg.AnnotationConfiguration`
* `Integrator` contract
* `Configuration` is  no longer `Serializable`
* `org.hibernate.dialect.Dialect.getQuerySequencesString` expected to retrieve catalog, schema, and increment values as well
* properties loaded from cfg.xml through EMF did not previously prefix names with "hibernate." this is now made consistent.

TODOs
=====
* Still need to go back and make all "persistent id generators" to properly implement ExportableProducer
* Add a setting to "consistently apply" naming strategies.  E.g. use the "join column" methods from hbm.xml binding.
* Along with this ^^ consistency setting, split the implicit naming strategy for join columns into multiple methods - one for each usage:
   * many-to-one
   * one-to-one
   * etc


Proposals for discussion
========================
* Currently there is a "post-binding" hook to allow validation of the bound model (PersistentClass,
Property, Value, etc).  However, the top-level entry points are currently the only possible place
(per contract) to throw exceptions when a validation fails".  I'd like to instead consider a pattern
where each level is asked to validate itself.  Given the current model, however, this is unfortunately
not a win-win situation.  `org.hibernate.boot.model.source.internal.hbm.ModelBinder#createManyToOneAttribute`
illustrates one such use case where this would be worthwhile, and also can illustrate how pushing the
validation (and exception throwing down) can be less than stellar given the current model.  In the process
of binding a many-to-one, we need to validate that any many-to-one that defines "delete-orphan" cascading
is a "logical one-to-one".  There are 2 ways a many-to-one can be marked as a "logical one-to-one"; first
is at the `<many-to-one/>` level; the other is through a singular `<column/>` that is marked as unique.
Occasionally the binding of the column(s) of a many-to-one need to be delayed until a second pass, which
means that sometimes we cannot perform this check immediately from the `#createManyToOneAttribute` method.
What would be ideal would be to check this after all binding is complete.  In current code, this could be
either an additional SecondPass or done in a `ManyToOne#isValid` override of `SimpleValue#isValid`.  The
`ManyToOne#isValid` approach illustrates the conundrum... In the `ManyToOne#isValid` call we know the real
reason the validation failed (non-unique many-to-one marked for orphan delete) but not the property name/path.
Simply returning false from `ManyToOne#isValid` would instead lead to a misleading exception message, which
would at least have the proper context to know the property name/path.
* Should `org.hibernate.boot.MetadataBuilder` be folded into `org.hibernate.boot.MetadataSources`?
* Consider an additional "naming strategy contract" specifically for logical naming.  This would be non-pluggable, and
would be the thing that generates the names we use to cross-reference and locate tables, columns, etc.