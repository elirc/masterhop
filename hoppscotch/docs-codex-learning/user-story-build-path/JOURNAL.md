# User Story Journal

## Why These Stories Were Chosen

The stories are grounded in Hoppscotch's actual domain: composing requests, saving metadata, inspecting history, working with teams, managing auth, syncing through GraphQL, and preserving versioned request data.

The safest stories start in the REST request UI (`packages/hoppscotch-common/src/components/http/Request.vue`) and nearby stores. The deeper stories cross the self-host sync layer and backend history module.

## Repeated Patterns

- Vue component state through `ref`, `computed`, `v-model`, and props.
- Custom stores through `DispatchingStore`.
- GraphQL wrappers in self-host platform packages.
- Backend resolver -> service -> Prisma.
- Versioned data schemas in `@hoppscotch/data`.
- Tests in Vitest for frontend services and Jest for backend services.

## Senior Watch Points

- Do not put platform-specific behavior in `hoppscotch-common`.
- Do not mutate request schema without a version/migration plan.
- Do not trust JSON strings at backend boundaries.
- Do not add backend mutations without auth and ownership enforcement.
- Do not let subscription echoes double-apply local changes.

