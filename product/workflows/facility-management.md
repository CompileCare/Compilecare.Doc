# Facility hierarchy management

## Purpose and scope

CompileCare models the physical hospital structure as:

`Tenant → Hospital → Department → Ward → Room → Bed`

The Web Facility workspace provides a single hierarchical administration surface for creating, reading, updating and deleting these records. It reuses the existing API and does not introduce a second client-side facility data model.

This workflow is facility administration only. It does not claim to implement bed occupancy, admission/discharge/transfer (ADT), nursing medication administration (MAR), room scheduling or clinical allocation. Those workflows require their own domain rules and acceptance criteria.

## Authorization model

The UI uses the effective membership permissions and scopes returned by the API, but the API remains the authorization authority for every operation.

| Permission | UI behavior |
|---|---|
| `facility.read` | Show the Facilities workspace and accessible hierarchy |
| `facility.write` | Show permitted create/update controls |
| `facility.delete` | Show delete controls; deletion still depends on server scope and child-record rules |

Additional scope rules applied by the workspace:

- Creating a hospital is shown only to a facility writer with tenant-wide scope.
- Editing the active hospital requires tenant-wide scope or a scope for that hospital.
- Creating a department requires tenant-wide or active-hospital management scope.
- Department updates/deletes and lower-level ward/room/bed operations are offered only for records returned within the caller's accessible parent scope.
- A missing UI control is not a security boundary. Every API request carries the bearer token and active `X-Tenant-ID`; server-side tenant filters, membership scopes and permission policies decide the result.

## API surface

All routes are tenant-scoped and require authorization.

| Resource | Collection | Detail |
|---|---|---|
| Hospitals | `GET/POST /api/v1/hospitals` | `GET/PUT/DELETE /api/v1/hospitals/{hospitalId}` |
| Departments | `GET/POST /api/v1/hospitals/{hospitalId}/departments` | `GET/PUT/DELETE /api/v1/hospitals/{hospitalId}/departments/{departmentId}` |
| Wards | `GET/POST /api/v1/departments/{departmentId}/wards` | `GET/PUT/DELETE /api/v1/departments/{departmentId}/wards/{wardId}` |
| Rooms | `GET/POST /api/v1/wards/{wardId}/rooms` | `GET/PUT/DELETE /api/v1/wards/{wardId}/rooms/{roomId}` |
| Beds | `GET/POST /api/v1/rooms/{roomId}/beds` | `GET/PUT/DELETE /api/v1/rooms/{roomId}/beds/{bedId}` |

The Web client sends the active tenant header for every request. It reloads child collections from the selected parent and clears descendant selections when the parent changes, preventing stale ward, room or bed selections from being reused against a different parent.

## Interaction and failure behavior

- The active hospital is selected through the shared workspace context.
- Departments, wards, rooms and beds are navigated in parent-to-child order.
- Create/update actions submit to the API and reload the affected collection; the UI does not optimistically assume a write succeeded.
- Delete actions require an explicit in-context confirmation.
- The API remains responsible for rejecting deletion when child facilities depend on the selected record. A conflict is shown without removing the item from the UI.
- Read-only memberships see the hierarchy but do not see mutation controls. Facility managers with write but no delete permission can create/update but cannot delete.
- The initial client collection request uses a page size of 100. Server-side pagination/search for very large facility collections remains a follow-up before large-scale production use.

## Data and privacy

- Facility administration does not require patient identity or clinical information.
- Use synthetic hospital, department, ward, room and bed records in development and acceptance tests.
- Never add real patient data, credentials or environment secrets to the repository or screenshots.

## Verification status

The Web implementation is tracked in [CompileCare.Web PR #34](https://github.com/CompileCare/CompileCare.Web/pull/34), stacked after the visual foundation and Reception/appointment/queue/clinical UI slices. The final status and exact hosted CI run IDs are recorded in the roadmap once the PR head checks complete. A real authenticated local-stack walkthrough remains required before marking the Antigravity facility-hierarchy scenario complete.
