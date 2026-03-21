.. _startup_performance_diagnosis:

***********************************************
pgAdmin Startup Performance Diagnosis (Desktop)
***********************************************

Scope
=====

This document analyzes why pgAdmin desktop startup can be slow, based on the
actual startup path in the codebase. It does **not** propose generic tuning;
it identifies concrete bottlenecks and practical, high-impact options.

Startup sequence (observed from code)
=====================================

Desktop startup is a two-process boot:

1. Electron process starts (``runtime/src/js/pgadmin.js``), chooses a port,
   spawns Python, then polls ``/misc/ping`` every 1 second until success.
2. Python process enters ``web/pgAdmin4.py`` -> ``create_app()`` in
   ``web/pgadmin/__init__.py`` and performs initialization before returning
   successful ping responses.

Key startup path references:

* Electron spawn and polling:
  ``runtime/src/js/pgadmin.js`` lines 267-344
* Python app creation:
  ``web/pgAdmin4.py`` lines 93-154
* Core initialization:
  ``web/pgadmin/__init__.py`` lines 223-763

Exact bottlenecks
=================

1) Synchronous schema migration and table validation (largest bottleneck)
---------------------------------------------------------------------------

Code references:

* migration orchestration:
  ``web/pgadmin/__init__.py`` lines 405-481
* migration execution:
  ``web/pgadmin/setup/db_upgrade.py`` line 25
* table-by-table validation:
  ``web/pgadmin/setup/db_table_check.py`` lines 18-36
* pre-migration SQLite full-file backup:
  ``web/pgadmin/__init__.py`` line 431

Why this is slow:

* Migration runs synchronously during app creation; ping cannot pass until done.
* SQLite migration path can copy the whole DB file before migration.
* ``check_db_tables()`` loops all metadata tables and performs per-table checks.
* There are 50 migration files under ``web/migrations/versions``; when version
  lag exists, upgrade work is serial and blocking.

Impact:

* This is the dominant startup delay on upgraded/large/slow-disk installs.
* On slow storage (network home folders, container volumes, antivirus-scanned
  paths), this can contribute the majority of total startup latency.


2) Eager blueprint/module loading during app creation
-----------------------------------------------------

Code references:

* static submodule imports:
  ``web/pgadmin/submodules.py`` lines 1-11
* registration loop:
  ``web/pgadmin/__init__.py`` lines 758-763

Why this is slow:

* Multiple feature-rich modules are imported/registered up front (e.g. browser,
  tools, llm, dashboard, settings, preferences, misc, authenticate).
* Importing these modules triggers additional imports, class construction, and
  route/blueprint registration before startup completes.

Impact:

* Adds fixed startup cost even when users only need a subset of features.
* Acts as a cold-start tax on every launch.


3) Driver and authentication registry eager loading
---------------------------------------------------

Code references:

* driver startup load:
  ``web/pgadmin/__init__.py`` line 552
* driver registry module imports:
  ``web/pgadmin/utils/driver/registry.py`` lines 17-29
* auth registry load:
  ``web/pgadmin/authenticate/__init__.py`` line 350
* auth registry module imports:
  ``web/pgadmin/authenticate/registry.py`` lines 17-39

Why this is slow:

* Registries eagerly import all supported modules at app init.
* This front-loads auth/driver initialization that may not be needed
  immediately.


4) Repeated synchronous configuration DB reads for security keys
----------------------------------------------------------------

Code references:

* ``web/pgadmin/__init__.py`` lines 497-503

Why this is slow:

* Three separate ORM queries are issued serially for startup keys
  (``CSRF_SESSION_KEY``, ``SECRET_KEY``, ``SECURITY_PASSWORD_SALT``).
* Small individually, but on high-latency or contended storage this is
  measurable and fully blocking.


5) Electron polling granularity adds avoidable wall-clock delay
----------------------------------------------------------------

Code references:

* polling interval:
  ``runtime/src/js/pgadmin.js`` lines 303-344

Why this is slow:

* Readiness checks are every 1000 ms; if backend becomes ready just after a
  poll, the UI can wait almost another second before launch proceeds.
* This introduces avoidable startup jitter and inflated average wait.


6) Synchronous runtime log writes in Electron startup path
----------------------------------------------------------

Code references:

* environment logging loop:
  ``runtime/src/js/pgadmin.js`` lines 241-265
* sync write implementation:
  ``runtime/src/js/misc.js`` lines 117-123

Why this is slow:

* Startup writes many log lines using ``fs.writeFileSync``.
* On some systems this becomes visible startup overhead due to sync disk I/O.


Other contributors and environment amplifiers
=============================================

* Slow/remote filesystem for profile directory or SQLite path magnifies all
  sync I/O bottlenecks (migration copy, table checks, sync logging).
* Antivirus/endpoint scanning can disproportionately penalize Python module
  imports and SQLite file operations.
* First run after install/upgrade is worst due to migrations and cold import
  caches.

Can startup be improved up to 10x?
==================================

Yes, in worst-case slow-start scenarios, approaching 10x is realistic. For
already-fast warm starts, the ceiling is lower.

High-impact changes required (not implemented here)
===================================================

1. Make heavy initialization incremental/lazy
---------------------------------------------

* Defer non-critical blueprint registration until first route access, or split
  app into a minimal core blueprint set plus on-demand modules.
* Defer non-critical registry initialization (drivers/auth providers) until
  first use.

Primary limit removed:
* Large fixed import/registration tax before first UI render.

Trade-off:
* First use of deferred feature incurs one-time latency.
* More complexity in lifecycle/error handling for deferred components.


2. Redesign migration strategy for desktop startup
--------------------------------------------------

* Keep startup path minimal when schema is already current.
* Avoid repeated or redundant validation calls.
* Minimize full-file copy behavior where safe.
* Optionally run migration preflight/upgrade as an explicit phase before full
  UI boot, with clear progress state.

Primary limit removed:
* Synchronous DB migration/validation dominating startup.

Trade-off:
* More complex migration state handling and UX for interrupted upgrades.
* Requires robust fallback to preserve reliability guarantees.


3. Reduce readiness detection latency
------------------------------------

* Replace fixed 1s polling with short initial interval + adaptive backoff, or
  event-driven readiness signaling from Python process.

Primary limit removed:
* 0-1 second avoidable wait introduced by coarse polling.

Trade-off:
* Slightly more orchestration complexity between Electron and backend.


4. Collapse small blocking operations
-------------------------------------

* Batch startup key retrieval into one DB query.
* Reduce sync log writes on the critical path.

Primary limit removed:
* Death-by-a-thousand-cuts from small serial blockers.

Trade-off:
* Modest code complexity; lower risk than migration/lazy-load redesign.


Practical expected gains
========================

If changes above are applied together:

* Slow cold starts (upgrade path, slow disk): highest gains, potentially near
  10x in worst cases.
* Typical cold starts: substantial gains, but often below 10x.
* Warm starts: noticeable but smaller gains; bounded by unavoidable base
  process startup and rendering costs.

Conclusion
==========

The main startup bottleneck is not a single micro-inefficiency; it is
front-loaded synchronous work in the critical path (especially migration and
eager module loading), plus coarse readiness polling. Significant improvement
requires shifting startup from "initialize everything before first window" to
"initialize core quickly, defer non-critical work safely."
