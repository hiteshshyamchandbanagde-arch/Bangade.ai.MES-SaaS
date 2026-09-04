# Production Module — Documentation Set

Core MES — Production Module, covering both **Discrete** and **Process** manufacturing, multi-tenant. Read in this order:

1. [Functional Design Document](./01-functional-design-document.md) — what the module must do, for whom
2. [Technical Design Document](./02-technical-design-document.md) — API design, service boundaries, tenancy enforcement
3. [Database Model](./03-database-model.md) — schema, ER diagram, RLS pattern
4. [Master Data Configuration](./04-master-data-configuration.md) — what tenants configure, in what order
5. [High-Level Architecture](./05-high-level-architecture.md) — system context, major components
6. [Detailed Architecture](./06-detailed-architecture.md) — state machines, sequence diagrams, calculation logic

**Status:** Draft — ready for stakeholder review before Figma design work begins. See each document's "open questions" section for items needing sign-off.

**Next step after review:** build the `ProductionOrderCard` component (Discrete/Process variants) in Figma, then the Production Overview screen, per the discussion that preceded these docs.
