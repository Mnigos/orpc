---
name: orpc-openapi
description: "Expose oRPC v2 procedures as a spec-compliant HTTP API: REST routing metadata, OpenAPIHandler, request coercion, OpenAPILink and restoring native response types, spec generation, and Scalar/Swagger docs. Use for that work, not merely because @orpc/openapi is installed; contract definition belongs to orpc-contract."
license: MIT
---

# oRPC over OpenAPI

oRPC procedures speak two protocols from one router: the RPC protocol (`RPCHandler`/`RPCLink`) and plain OpenAPI HTTP (`OpenAPIHandler`/`OpenAPILink`). This skill covers the OpenAPI side. For core builder, middleware, context, and client concepts, load the `orpc` skill; for contract-first workflows with `@orpc/contract`, load the `orpc-contract` skill.

This skill targets oRPC v2. Check the exact installed `@orpc/openapi` and `@orpc/server` versions yourself, prerelease suffix included, from the lockfile or `node_modules/@orpc/openapi/package.json` (or the project's package manager, such as `pnpm ls @orpc/openapi`), without loading the `orpc` skill just for that. A 1.x install means the v1 docs at https://v1.orpc.dev apply and this skill does not; upgrading is a separate, requested migration (`orpc-migrate` skill), never a side effect. If installing or upgrading is in scope, check the registry's dist-tags at that moment rather than assuming 2.x is still on `beta`, and use the project's package manager. Pretrained oRPC knowledge is often v1-shaped (`.route`/`.prefix` on the builder, `@orpc/openapi-client`): when unsure of any API below, fetch the docs page matching the installed version first (see [Full documentation](#full-documentation)). Zod and the Fetch adapter below are illustrative; keep the project's schema library, converter, runtime adapter, and code-first or contract-first style.

## Routing

A procedure defaults to `POST` with a path derived from the router structure (`planet.create` becomes `POST /planet/create`). Override with `openapi` metadata, on server (`os`) and contract (`oc`) builders alike:

```ts
import { openapi } from '@orpc/openapi'
import { os } from '@orpc/server'
import { z } from 'zod'

const getPlanet = os
  .meta(openapi({ method: 'GET', path: '/planets/{id}', successStatus: 200 }))
  .input(z.object({ id: z.string() }))
  .handler(async ({ input }) => ({ id: input.id, name: 'Earth' }))
```

- Path params: put `{id}` in `path` and the same key as a required field in the input schema. Use `{+path}` for catch-all segments that may contain `/`.
- `prefix` prepends a path to a procedure or a whole router: `os.meta(openapi({ prefix: '/api/v2' })).router({...})`. Always set a `prefix` on lazy routers so lazy loading only triggers for relevant requests.
- Merging across repeated `.meta(openapi(...))` calls: `prefix` and `tags` concatenate, `method`/`path`/`successStatus` last-wins, and setting a field to `undefined` resets it to the default.
- `successStatus` defaults to `200` and must be below 400.

For direct routing without `.meta(openapi(...))`, enable the `.route` extension with a side-effect import in a module that always runs at startup (base builder file or server entry):

```ts
import '@orpc/openapi/extensions/route'

const ping = os.route({ method: 'GET', path: '/ping' }).handler(async () => 'pong')
```

## Input and output mapping

Compact mode (default): path params merge with query params or the request body depending on the HTTP method, so `GET /planets/earth?q=life` yields input `{ id: 'earth', q: 'life' }`. The handler's return value becomes the response body with status `successStatus`.

Set `inputStructure: 'detailed'` in the metadata to receive `{ params, query, headers, body }` instead (define only the fields you need in the input schema). Set `outputStructure: 'detailed'` to return `{ status?, headers?, body? }` and vary the status per response.

Query strings and form data are decoded with bracket notation (fetch `openapi/bracket-notation` before designing schemas for nested query input; its limits are not guessable):

- Repeated keys become arrays: `?color=red&color=blue` gives `['red', 'blue']`; `color[]=red` pushes too.
- `[number]` targets an array index; `[key]` targets an object property: `?filter[status]=active` gives `{ filter: { status: 'active' } }`.
- It cannot represent empty objects or arrays, root-level arrays, or objects whose keys are all numbers, and query/form values always arrive as strings (or files, in form data).

Override decoding per parameter with `paramsStyles` (`'primitive'`, `'comma-delimited-array'`, `'comma-delimited-object'`) and `queryStyles` (those plus `'array'`, `'json'`, and space/pipe-delimited variants). Use `requestBodyHint`/`responseBodyHint` (`'json'`, `'form-data'`, `'event-stream'`, `'octet-stream'`, `'file'`, ...) when headers alone cannot tell the parser how to handle the body, for example a raw `ReadableStream` upload.

## Serving: OpenAPIHandler

Import from the adapter subpath (`@orpc/openapi/fetch`, `@orpc/openapi/node`, ...):

```ts
import { SmartCoercionHandlerPlugin } from '@orpc/json-schema'
import { OpenAPIHandler } from '@orpc/openapi/fetch'
import { ZodToJsonSchemaConverter } from '@orpc/zod'

const handler = new OpenAPIHandler(router, {
  plugins: [
    new SmartCoercionHandlerPlugin({ converters: [new ZodToJsonSchemaConverter()] }),
  ],
})

export async function fetch(request: Request): Promise<Response> {
  const { matched, response } = await handler.handle(request, { prefix: '/api', context: {} })
  return matched ? response : new Response('Not Found', { status: 404 })
}
```

`OpenAPIHandler` coexists with `RPCHandler`: both accept the same router, so mount them on different prefixes (for example `/api` and `/rpc`) and try each in turn, returning the first `matched` response.

Smart Coercion: query, path, and form values arrive as strings, so add `SmartCoercionHandlerPlugin` whenever input schemas expect non-string types from those sources. It coerces schema-driven, lossless conversions only (`'123'` to `123`, `'true'`/`'on'` to `true`, ISO strings to `Date`, arrays to `Set`/`Map` via `x-native-type`) and leaves ambiguous values untouched. Skip it when you already coerce in the schema or performance is critical; it adds runtime overhead. This is request-input coercion on the handler only: it never changes what the client receives back.

Other handler options: `interceptors`/`routingInterceptors`/`clientInterceptors` for logging and error mapping, `filter` to exclude procedures from matching, and `errorStatusMap` plus `customErrorResponseBodyEncoder` to customize error responses (by default `ORPCError` codes map to statuses via `COMMON_ERROR_STATUS_MAP`, for example `NOT_FOUND` to 404).

## Calling: OpenAPILink

`OpenAPILink` calls an OpenAPI-shaped oRPC API (or any spec-compliant server) through a typesafe client. Unlike `RPCLink`, it takes the contract (or an unlazied router) as a runtime value, not a type, to read each procedure's method and path:

```ts
import type { RouterContractClient } from '@orpc/contract'
import type { JsonifiedClient } from '@orpc/openapi'
import { createORPCClient } from '@orpc/client'
import { OpenAPILink } from '@orpc/openapi/fetch'

const link = new OpenAPILink(contract, {
  origin: 'https://api.example.com',
  url: '/api',
  headers: ({ context }) => ({
    authorization: context?.token ? `Bearer ${context.token}` : undefined,
  }),
})

const client: JsonifiedClient<RouterContractClient<typeof contract>> = createORPCClient(link)
```

With a router instead of a contract, type the client as `JsonifiedClient<RouterClient<typeof router>>` (`RouterClient` from `@orpc/server`). In a browser bundle the runtime value must come from a shared contract package, a minified JSON contract, or a build-time macro; never import the server's router implementation into client code. The minify flow is in the `orpc-contract` skill under "Ship the contract", the macro flow in `openapi/link-without-runtime-imports`.

### Native types on responses

`JsonifiedClient` exists because OpenAPI serialization is one-way: a `Date` leaves the server as an ISO string and stays a string on the client. Declaring `z.date()` in `.output`, or coercing on the handler side, changes nothing for the client, because output validation runs on the server before serialization. Keep `JsonifiedClient` unless the link itself re-parses responses, with one of two plugins:

- `ResponseValidationLinkPlugin(contract)` from `@orpc/contract/plugins` runs the contract's `.output` (and error `data`) schemas on the client. With a coercing output schema (`z.coerce.date<Date>()`, `z.coerce.bigint<bigint>()`) it turns the JSON form back into the native type; the coercion rules stay explicit in the schema. It only works with schemas that accept the wire form: a transform whose output the schema cannot accept as input fails on the client, because the server already sent the transformed value. Docs: `plugins/response-validation`.
- `SmartCoercionLinkPlugin(contract, { converters: [...] })` from `@orpc/json-schema` derives lossless conversions from the schemas' JSON Schema (`x-native-type`), so no coercion needs to be written into the schema. Docs: `plugins/smart-coercion`.

Both read schemas from the contract you pass, so a minified JSON contract (`minifyRouterContract` strips schemas) cannot drive either; give the link the real contract with schemas or keep `JsonifiedClient`. Drop the `JsonifiedClient` wrapper only once the plugin and schemas actually restore every declared native shape the client relies on, output and error `data` fields included; the plugin's presence alone does not make the wrapper wrong. Handler-side Smart Coercion covers request input; link-side plugins cover responses; there is no blanket rule to use or avoid `z.coerce.date`, pick it per side and per schema. Both plugins can only restore types the OpenAPI serializer can represent: `openapi/expanding-type-support-for-link`.

## Spec document and interactive docs

`OpenAPIGenerator` turns a router or contract into an OpenAPI 3.2 document (pass any `3.1.x` or `3.0.x` `version` for tools that stop at an older version; `QUERY` procedures need 3.2). `OpenAPIReferenceHandlerPlugin` serves the spec at `/spec.json` and a Scalar UI at `/` under the handler prefix (change with `specPath`/`docsPath`, or set `provider: 'swagger'` for Swagger UI):

```ts
import { OpenAPIGenerator } from '@orpc/openapi'
import { OpenAPIReferenceHandlerPlugin } from '@orpc/openapi/plugins'
import { ZodToJsonSchemaConverter } from '@orpc/zod'

const generator = new OpenAPIGenerator({ converters: [new ZodToJsonSchemaConverter()] })

const handler = new OpenAPIHandler(router, {
  plugins: [
    new OpenAPIReferenceHandlerPlugin({
      spec: () => generator.generate(router, {
        base: {
          info: { title: 'Planet API', version: '1.0.0' },
          servers: [{ url: '/api' }], // absolute URL in production
        },
      }),
    }),
  ],
})
```

Enrich the document through `openapi` metadata: `operationId`, `summary`, `description`, `tags`, `successDescription`, and a `spec` callback that receives the generated operation object and returns an extended one (security requirements, extra responses). Write `spec` and `base` as OpenAPI 3.2 objects even when generating 3.1 or 3.0; the generator downgrades the whole document. Converters also exist for Valibot (`@orpc/valibot`) and ArkType (`@orpc/arktype`); schemas without a matching converter fall back to Standard JSON Schema conversion.

Verify the wiring before declaring success with the project's existing checks and a test through the real `OpenAPIHandler` (and `OpenAPILink`, if you touched the client side): request one routed endpoint, plus `/spec.json` under the handler prefix when the document matters, and confirm the method, path, status, and body shape you configured. A direct `call` of the procedure bypasses routing, input mapping, and serialization, so it proves nothing about the HTTP side.

## Contract-first

Contract-first workflows belong to the `orpc-contract` skill: defining the shape with `oc` from `@orpc/contract`, implementing it with `implement` from `@orpc/server`, consuming the contract from clients, shipping it as minified JSON, and generating it from an existing OpenAPI spec with Hey API. On the OpenAPI side a contract behaves exactly like a router: attach routes with `.meta(openapi({...}))` on `oc` as shown in [Routing](#routing), serve the implemented router with `OpenAPIHandler` as usual, generate the spec document from the contract directly, and give `OpenAPILink` the contract as its runtime value.

## Full documentation

This skill is a summary and v2 is still moving. The docs are served at https://orpc.dev (the v1 docs stay at https://v1.orpc.dev) and describe the latest release. Match retrieval to the installed or target version: for an older 2.x, especially a beta, read the same page from the release tag at https://github.com/middleapi/orpc/tree/v<version> (locate it under the docs content there, since the docs layout and the `.md`/`.mdx` extension vary across releases; the package source is also there) or the installed package's `.d.ts` files in `node_modules`. The installed version's types and tagged docs win over the live page, and the live page wins over this skill.

- https://orpc.dev/llms.txt : index of every page with descriptions
- https://orpc.dev/llms-full.txt : the entire docs in one file (large; prefer single pages)
- Append `.md` to any docs URL for that page's exact source markdown (for example https://orpc.dev/docs/openapi/routing.md)

Pages to fetch when you need details beyond this skill:

- OpenAPI: `openapi/routing`, `openapi/input-and-output-mapping`, `openapi/bracket-notation`, `openapi/serializer`, `openapi/handler`, `openapi/link`, `openapi/expanding-type-support-for-link`, `openapi/link-without-runtime-imports`, `openapi/specification`, `openapi/scalar`
- Plugins: `plugins/smart-coercion`, `plugins/response-validation`, `plugins/openapi-reference`
