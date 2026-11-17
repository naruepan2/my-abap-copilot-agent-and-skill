---
name: rap-lifecycle
description: The ordered create-and-publish chain for a RAP business object: which server owns each step, and the checks that stop a half-published service. Use when building a RAP stack, adding a layer (table, CDS entity, projection, behavior definition, service definition, service binding), or publishing an OData service. Not for editing the logic inside a behavior implementation that already exists.
---

# RAP lifecycle

Follow this chain in order. Each step names the owning server. If a step's server is unavailable,
stop and report rather than skipping the step — a stack that skipped a layer fails later, somewhere
less obvious.

## The chain

1. **Package** — `SAPManage(action="create_package")`.

2. **Table, CDS entity, projection, service definition** — ADT MCP
   `abap_objects_generator_run`, which derives its schema from the target system's release. Fall
   back to `SAPWrite(action="batch_create")` in dependency order when the generator is unavailable.

3. **Behavior definition** — `SAPWrite(type="BDEF")`. The deterministic RAP preflight runs
   automatically and blocks malformed definitions; do not disable it to force a write through.

4. **Behavior pool** — `SAPWrite(action="generate_behavior_implementation", dryRun=true)` to review
   the plan, then repeat without `dryRun`. This creates missing `lhc_<alias>` skeletons and injects
   every required handler signature; it is the MCP equivalent of ADT's "Generate Behavior
   Implementation" quick-fix.

5. **Service binding** — `SAPWrite(type="SRVB", odataVersion="V4", category=...)`. ABAP Remote FS
   cannot create V4 bindings, so do not attempt it there. ARC-1 and ABAP FS document the `category`
   codes in opposite order, so verify the resulting Binding Category in ADT after the first create in
   any system.

6. **Activation** — `SAPActivate(objects=[...])` in dependency order, or
   `batch_create(activateAtEnd=true)` when siblings cross-reference each other.

7. **Publication** — `SAPActivate(action="publish_srvb", service_type="odatav4")`.

8. **Impact analysis before changing a CDS field** — `SAPContext(action="impact")`, which buckets
   consumers into RAP artifacts instead of raw where-used rows.

9. **Verification** — ADT MCP `abap_business_service_list` for the published contract, ABAP Unit for
   behavior, ATC for quality.

## Report each step's actual outcome

A saved object is not an activated object, and an activated object is not a published service. Those
three states are distinct and the difference is where most "it works" claims go wrong.

## Before starting

The namespace rule, the write gates, and the pre-flight confirmation are in the agent file and are
not relaxed by anything here. A RAP scaffold is a create, so it needs the confirmed system and
client first, and every object name must begin with `Z` or `Y`.
