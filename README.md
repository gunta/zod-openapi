<p align="center">
  <img src="zod-openapi.svg" width="200px" align="center" alt="zod-openapi logo" />
  <h1 align="center">zod-openapi</h1>
</p>
<p align="center">
A Typescript library to use <a href="https://github.com/colinhacks/zod">Zod</a> Schemas to create OpenAPI v3.x documentation
</p>
<div align="center">
<a href="https://www.npmjs.com/package/zod-openapi"><img src="https://img.shields.io/npm/v/zod-openapi"/><a>
<a href="https://www.npmjs.com/package/zod-openapi"><img src="https://img.shields.io/npm/dm/zod-openapi"/><a>
<a href="https://nodejs.org/en/"><img src="https://img.shields.io/badge/node-%3E%3D%2016.11-brightgreen"/><a>
<a href="https://github.com/samchungy/zod-openapi/actions/workflows/test.yml"><img src="https://github.com/samchungy/zod-openapi/actions/workflows/test.yml/badge.svg"/><a>
<a href="https://github.com/samchungy/zod-openapi/actions/workflows/release.yml"><img src="https://github.com/samchungy/zod-openapi/actions/workflows/release.yml/badge.svg"/><a>
<a href="https://github.com/seek-oss/skuba"><img src="https://img.shields.io/badge/🤿%20skuba-powered-009DC4"/><a>
</div>
<br>

## Install

Install via `npm` or `yarn`:

```bash
npm install zod zod-openapi
## or
yarn add zod zod-openapi
```

## Usage

### `extendZodWithOpenApi`

This mutates Zod to add an extra `.openapi()` method. Make a side-effectful import at the top of your entry point(s).

```typescript
import { z } from 'zod';
import { extendZodWithOpenApi } from 'zod-openapi';

extendZodWithOpenApi(z);

z.string().openapi({ description: 'hello world!', example: 'hello world' });
```

#### `.openapi()`

Use the `.openapi()` method to add metadata to a specific Zod type. The `.openapi()` method takes an object with the following options:

|     Option      |                                                         Description                                                         |
| :-------------: | :-------------------------------------------------------------------------------------------------------------------------: |
| OpenAPI Options | This will take any option you would put on a [SchemaObject](https://swagger.io/docs/specification/data-models/data-types/). |
|  `effectType`   |                             Use to override the creation type for a [Zod Effect](#zod-effects)                              |
|     `param`     |                                Use to provide metadata for [request parameters](#parameters)                                |
|    `header`     |                              Use to provide metadata for [response headers](#response-headers)                              |
|      `ref`      |                                 Use this to [auto register a schema](#creating-components)                                  |
|    `refType`    |                 Use this to set the creation type for a component which is not referenced in the document.                  |
|     `type`      |                 Use this to override the generated type. If this is provided no metadata will be generated.                 |
|  `unionOneOf`   |                           Set to `true` to force a ZodUnion to output `oneOf` instead of `allOf`                            |

### `createDocument`

Creates an OpenAPI documentation object

```typescript
import { z } from 'zod';
import { createDocument, extendZodWithOpenApi } from 'zod-openapi';

extendZodWithOpenApi(z);

const jobId = z.string().openapi({
  description: 'Job ID',
  example: '12345',
});

const title = z.string().openapi({
  description: 'Job title',
  example: 'My job',
});

const document = createDocument({
  openapi: '3.1.0',
  info: {
    title: 'My API',
    version: '1.0.0',
  },
  paths: {
    '/jobs/{jobId}': {
      put: {
        requestParams: { path: z.object({ jobId }) },
        requestBody: {
          content: {
            'application/json': { schema: z.object({ title }) },
          },
        },
        responses: {
          '200': {
            description: '200 OK',
            content: {
              'application/json': { schema: z.object({ jobId, title }) },
            },
          },
        },
      },
    },
  },
});
```

Generates the following object:

```json
{
  "openapi": "3.1.0",
  "info": {
    "title": "My API",
    "version": "1.0.0"
  },
  "paths": {
    "/jobs/{jobId}": {
      "put": {
        "parameters": [
          {
            "in": "path",
            "name": "jobId",
            "schema": {
              "type": "string",
              "description": "Job ID",
              "example": "12345"
            }
          }
        ],
        "requestBody": {
          "content": {
            "application/json": {
              "schema": {
                "type": "object",
                "properties": {
                  "title": {
                    "type": "string",
                    "description": "Job title",
                    "example": "My job"
                  }
                },
                "required": ["title"]
              }
            }
          }
        },
        "responses": {
          "200": {
            "description": "200 OK",
            "content": {
              "application/json": {
                "schema": {
                  "type": "object",
                  "properties": {
                    "jobId": {
                      "type": "string",
                      "description": "Job ID",
                      "example": "12345"
                    },
                    "title": {
                      "type": "string",
                      "description": "Job title",
                      "example": "My job"
                    }
                  },
                  "required": ["jobId", "title"]
                }
              }
            }
          }
        }
      }
    }
  }
}
```

### Request Parameters

Query, Path, Header & Cookie parameters can be created using the `requestParams` key under the `method` key as follows:

```typescript
createDocument({
  paths: {
    '/jobs/{a}': {
      put: {
        requestParams: {
          path: z.object({ a: z.string() }),
          query: z.object({ b: z.string() }),
          cookie: z.object({ cookie: z.string() }),
          header: z.object({ 'custom-header': z.string() }),
        },
      },
    },
  },
});
```

If you would like to declare parameters in a more traditional way you may also declare them using the [parameters](https://swagger.io/docs/specification/describing-parameters/) key. The definitions will then all be combined.

```ts
createDocument({
  paths: {
    '/jobs/{a}': {
      put: {
        parameters: [
          z.string().openapi({
            param: {
              name: 'job-header',
              in: 'header',
            },
          }),
        ],
      },
    },
  },
});
```

### Request Body

Where you would normally declare the [media type](https://swagger.io/docs/specification/media-types/), set the `schema` as your Zod Schema as follows.

```typescript
createDocument({
  paths: {
    '/jobs': {
      get: {
        requestBody: {
          content: {
            'application/json': { schema: z.object({ a: z.string() }) },
          },
        },
      },
    },
  },
});
```

If you wish to use OpenAPI syntax for your schemas, simply add an OpenAPI schema to the `schema` field instead.

### Responses

Similarly to the [Request Body](#request-body), simply set the `schema` as your Zod Schema as follows. You can set the response headers using the `headers` key.

```typescript
createDocument({
  paths: {
    '/jobs': {
      get: {
        responses: {
          200: {
            description: '200 OK',
            content: {
              'application/json': { schema: z.object({ a: z.string() }) },
            },
            headers: z.object({
              'header-key': z.string(),
            }),
          },
        },
      },
    },
  },
});
```

### Creating Components

OpenAPI allows you to define reusable [components](https://swagger.io/docs/specification/components/) and this library allows you to replicate that in two separate ways.

1. Auto registering schema
2. Manually registering schema

#### Schema

If we take the example in `createDocument` and instead create `title` as follows

##### Auto Registering Schema

```typescript
const title = z.string().openapi({
  description: 'Job title',
  example: 'My job',
  ref: 'jobTitle', // <- new field
});
```

Wherever `title` is used in schemas across the document, it will instead be created as a reference.

```json
{ "$ref": "#/components/schemas/jobTitle" }
```

`title` will then be outputted as a schema within the components section of the documentation.

```json
{
  "components": {
    "schemas": {
      "jobTitle": {
        "type": "string",
        "description": "Job title",
        "example": "My job"
      }
    }
  }
}
```

This can be an extremely powerful way to generate better Open API documentation. There are some Open API features like [discriminator mapping](https://swagger.io/docs/specification/data-models/inheritance-and-polymorphism/) which require all schemas in the union to contain a ref.

##### Manually Registering Schema

Another way to register schema instead of adding a `ref` is to add it to the components directly. This will still work in the same way as `ref`. So whenever we run into that Zod type we will replace it with a reference.

eg.

```typescript
createDocument({
  components: {
    schemas: {
      jobTitle: title, // this will register this Zod Schema as jobTitle unless `ref` in `.openapi()` is specified on the type
    },
  },
});
```

##### Zod Effects

`.transform()` and `.pipe()` are complicated because they technically comprise of two types (input & output). This means that we need to understand which type you are creating. In particular with transform it is very difficult to infer the output type. This library will automatically select which _type_ to use by checking how the schema is used based on the following rules:

_Input_: Request Bodies, Request Parameters, Headers

_Output_: Responses, Response Headers

If a registered schema with a transform or pipeline is used in both a request and response schema you will receive an error because the created schema for each will be different. To override the creation type for a specific ZodEffect, add an `.openapi()` field on it and set the `effectType` field to `input` or `output`. This will force this library to always generate the input/output type even if we are creating a response (output) or request (input) type. You typically want to use this when you know your transform has not changed the type.

`.preprocess()` will always return the `output` type even if we are creating an input schema. If a different input type is required you can achieve this with a `.transform()` combined with a `.pipe()` or simply declare a manual `type` in `.openapi()`.

If you are adding a ZodSchema directly to the `components` section which is not referenced anywhere in the document, additional context may be required to create either an input or output schema. You can do this by setting the `refType` field to `input` or `output` in `.openapi()`. This defaults to `output` by default.

#### Parameters

Query, Path, Header & Cookie parameters can be similarly registered:

```typescript
// Easy auto registration
const jobId = z.string().openapi({
  description: 'Job ID',
  example: '1234',
  param: { ref: 'jobRef' },
});

createDocument({
  paths: {
    '/jobs/{jobId}': {
      put: {
        requestParams: {
          header: z.object({
            jobId,
          }),
        },
      },
    },
  },
});

// or more verbose auto registration
const jobId = z.string().openapi({
  description: 'Job ID',
  example: '1234',
  param: { in: 'header', name: 'jobId', ref: 'jobRef' },
});

createDocument({
  paths: {
    '/jobs/{jobId}': {
      put: {
        parameters: [jobId],
      },
    },
  },
});

// or manual registeration
const otherJobId = z.string().openapi({
  description: 'Job ID',
  example: '1234',
  param: { in: 'header', name: 'jobId' },
});

createDocument({
  components: {
    parameters: {
      jobRef: jobId,
    },
  },
});
```

#### Response Headers

Response headers can be similarly registered:

```typescript
const header = z.string().openapi({
  description: 'Job ID',
  example: '1234',
  header: { ref: 'some-header' },
});

// or

const jobIdHeader = z.string().openapi({
  description: 'Job ID',
  example: '1234',
});

createDocument({
  components: {
    headers: {
      someHeaderRef: jobIdHeader,
    },
  },
});
```

#### Responses

Entire Responses can also be registered

```typescript
const response: ZodOpenApiResponseObject = {
  description: '200 OK',
  content: {
    'application/json': {
      schema: z.object({ a: z.string() }),
    },
  },
  ref: 'some-response',
};

//or

const response: ZodOpenApiResponseObject = {
  description: '200 OK',
  content: {
    'application/json': {
      schema: z.object({ a: z.string() }),
    },
  },
};

createDocument({
  components: {
    responses: {
      'some-response': response,
    },
  },
});
```

## Supported OpenAPI Versions

Currently the following versions of OpenAPI are supported

- `3.0.0`
- `3.0.1`
- `3.0.2`
- `3.0.3`
- `3.1.0`

Setting the `openapi` field will change how the some of the components are rendered.

```ts
createDocument({
  openapi: '3.1.0',
});
```

For example in `z.string().nullable()` will be rendered differently

`3.0.0`

```json
{
  "type": "string",
  "nullable": true
}
```

`3.1.0`

```json
{
  "type": ["string", "null"]
}
```

## Supported Zod Schema

- ZodAny
- ZodArray
  - `minItems`/`maxItems` mapping for `.length()`, `.min()`, `.max()`
- ZodBoolean
- ZodBranded
- ZodCatch
- ZodDate
  - `string` `type` mapping by default
- ZodDefault
- ZodDiscriminatedUnion
  - `discriminator` mapping when all schemas in the union contain a `ref`.
- ZodEffects
  - `transform` support for request schemas. See [Zod Effects](#zod-effects) for how to enable response schema support
  - `pre-process` support. We assume that the input type is the same as the output type. Otherwise pipe and transform can be used instead.
  - `refine` full support
- ZodEnum
- ZodLazy
  - The recursive schema within the ZodLazy or the ZodLazy _**must**_ be registered as a component. See [Creating Components](#creating-components) for more information.
- ZodLiteral
- ZodNativeEnum
  - supporting `string`, `number` and combined enums.
- ZodNull
- ZodNullable
- ZodNumber
  - `integer` `type` mapping for `.int()`
  - `exclusiveMin`/`min`/`exclusiveMax`/`max` mapping for `.min()`, `.max()`, `lt()`, `gt()`
- ZodObject
  - `additionalProperties` mapping for `.catchall()`, `.strict()`
  - `allOf` mapping for `.extend()` when the base object is registered and does not have `catchall()`, `strict()` and extension does not override a field.
- ZodOptional
- ZodPipeline
  - See [Zod Effects](#zod-effects) for more information.
- ZodReadonly
- ZodRecord
- ZodSet
  - Treated as an array with `uniqueItems` (you may need to add a pre-process)
- ZodString
  - `format` mapping for `.url()`, `.uuid()`, `.email()`, `.datetime()`
  - `minLength`/`maxLength` mapping for `.length()`, `.min()`, `.max()`
  - `pattern` mapping for `.regex()`, `.startsWith()`, `.endsWith()`, `.includes()`
- ZodTuple
  - `items` mapping for `.rest()`
  - `prefixItems` mapping for OpenAPI 3.1.0+
- ZodUnion
  - By default it outputs an `allOf` schema. Use `unionOneOf` to change this to output `oneOf` instead.
- ZodUnknown

If this library cannot determine a type for a Zod Schema, it will throw an error. To avoid this, declare a manual `type` in the `.openapi()` section of that schema.

eg.

```typescript
z.custom().openapi({ type: 'string' });
```

## Examples

See the library in use in the [examples](./examples/) folder.

- Simple - [setup](./examples/simple/createSchema.ts) | [openapi.yml](./examples/simple/openapi.yml) | [redoc documentation](https://samchungy.github.io/zod-openapi/examples/simple/redoc-static.html)

## Ecosystem

- [fastify-zod-openapi](https://github.com/samchungy/fastify-zod-openapi) - Fastify plugin for zod-openapi. This includes type provider, Zod schema validation, Zod schema serialization and Swagger UI support.

- [eslint-plugin-zod-openapi](https://github.com/samchungy/eslint-plugin-zod-openapi) - Eslint rules for zod-openapi. This includes features which can autogenerate Typescript comments for your Zod types based on your `description`, `example` and `deprecated` fields.

## Comparisons

### [@asteasolutions/zod-to-openapi](./docs/comparisons.md)

## Development

### Prerequisites

- Node.js LTS
- Yarn 1.x

```shell
yarn
yarn build
```

### Test

```shell
yarn test
```

### Lint

```shell
# Fix issues
yarn format

# Check for issues
yarn lint
```

### Release

To release a new version

1. Create a [new GitHub Release](https://github.com/samchungy/zod-openapi/releases/new)
2. Select `🏷️ Choose a tag`, enter a version number. eg. `v1.2.0` and click `+ Create new tag: vX.X.X on publish`.
3. Click the `Generate release notes` button and adjust the description.
4. Tick the `Set as the latest release` box and click `Publish release`. This will trigger the `Release` workflow.
5. Check the `Pull Requests` tab for a PR labelled `Release vX.X.X`.
6. Click `Merge Pull Request` on that Pull Request to update master with the new package version.

To release a new beta version

1. Create a [new GitHub Release](https://github.com/samchungy/zod-openapi/releases/new)
2. Select `🏷️ Choose a tag`, enter a version number with a `-beta.X` suffix eg. `v1.2.0-beta.1` and click `+ Create new tag: vX.X.X-beta.X on publish`.
3. Click the `Generate release notes` button and adjust the description.
4. Tick the `Set as a pre-release` box and click `Publish release`. This will trigger the `Prerelease` workflow.
