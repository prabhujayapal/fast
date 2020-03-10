# FAST Tooling

FAST Tooling is a library agnostic specific set of utilities to assist in creating web UI.

![JavaScript](https://img.shields.io/badge/ES6-Supported-yellow.svg?style=for-the-badge&logo=JavaScript) &nbsp; ![TypeScript](https://img.shields.io/badge/TypeScript-Supported-blue.svg?style=for-the-badge)

- [Installation](#installation)
- [Concepts](#concepts)
    - [JSON Schema](#json-schema)
        - [Nesting data](#nesting-data)
- [Message system](#message-system)
    - [Sending and receiving messages](#sending-and-receiving-messages)
- [Data utilities](#data-utilities)
    - [Generating data](#generating-data-from-json-schema)
    - [Mapping data](#mapping-data)

## Installation

`npm i --save @microsoft/fast-tooling`

## Concepts

### JSON Schema

[JSON schema](http://json-schema.org/) are used by FAST tooling libraries for generating data and creating UI. They have been extended to provide additional hooks for plugin systems in the FAST tooling libraries.

#### Nesting data

To identify nesting structures in the JSON schemas, such as with composable components, use the `linkedDataSchema` export from the `@microsoft/fast-tooling` package which defines the interface expected for the link and adds a property key to identify this section of data as linked data.

Example JSON Schema with linked data properties:

```ts
import { linkedDataSchema } from "@microsoft/fast-tooling";

export default {
    $schema: "http://json-schema.org/schema#",
    title: "Component with nested properties",
    type: "object",
    id: "nestable-component",
    properties: {
        children: {
            ...linkedDataSchema,
            title: "Children",
            type: "string"
        },
    }
}
```

Although JSON schema can be written in JSON, it is recommended creating the schema as a data blob in a JavaScript or TypeScript file so that it can use the provided helper exports.

## Message system

FAST tooling components rely on including a secondary script which contains a [web worker](https://developer.mozilla.org/en-US/docs/Web/API/Worker) called the message system.

This worker performs all of the data manipulation and provides a navigational data structure based on data passed.

### Sending and receiving messages

There is a secondary export, `MessageSystem`, which must be instantiated with the location on the server of the web worker. The file is located at `@microsoft/fast-tooling/message-system.min.js`. This is then passed to various components that are part of the tooling package to sync data and navigation.

Example implementation:
```js
import { MessageSystem } from "@microsoft/fast-tooling";

let fastMessageSystem;

// Your JSON schema
const mySchema = {
    id: "my-schema",
    type: "object",
    properties: {
        foo: {
            type: "string"
        }
    }
}

if (window.Worker) {

    fastMessageSystem = new MessageSystem({
        // the location of the file on the server
        // if you are using webpack, include it in the entry section of the config
        webWorker: "message-system.min.js",

        // your data dictionary to initialize with (you may only need a single item)
        dataDictionary: [
            {
                dataDictionaryKey1: {
                    schemaId: mySchema.id,
                    data: {
                        foo: "Hello world"
                    },
                },
            },
            "dataDictionaryKey1",
        ],

        // your dictionary of schemas to validate data in the dictionary
        schemas: {
            [mySchema.id]: mySchema,
        },
    });
}
```

## Data utilities

Data utilities are used for the various data manipulations in the message system, they are also provided as exports.

### Generating data from JSON schema

Data may be generated from a JSON schema using the `getDataFromSchema` export. This will only generate the required items as dictated by the JSON schema, and will always choose the first potential match in any situation, for example if a property is an enum and is required, it will add the first value in the enum list.

An example of generating data from the `@microsoft/fast-tooling` package:

```javascript
import { getDataFromSchema } from "@microsoft/fast-tooling";

const data = getDataFromSchema(schema);
```

### Mapping data

Data from the dictionary of data can be mapped with a helper `mapDataDictionary`. This will allow you transform the data dictionary into another type of data structure by writing a helper. For example, if the data dictionary represented React component props, you could write a mapper using the React createElement function and return a functional React component.

The `mapDataDictionary` export takes an object with the following properties:

- **dataDictionary** - A dictionary of data, this is similar to other data dictionary formats in this library but instead of specifying a root data item, it is the dictionary only.
- **dataDictionaryKey** - This should be the root data items key.
- **schemaDictionary** - This should be the dictionary of JSON schemas where each schema is identified in the object by its id which is used as a key.
- **mapper** - The function provided that maps the data.

The mapping function that should be provided as the **mapper** in the `mapDataDictionary` argument accepts as its argument an object with the following properties:

- **data** - The raw unchanged data.
- **resolvedData** - Data that has been run through the mapper before.
- **schema** - The JSON schema that maps to this piece of data, it should validate against the **data** property.

Example:

```javascript
import { mapDataDictionary } from "@microsoft/fast-tooling";

const mappingFunction = function(config) {
    return config.resolvedData;
}

const mappedData = mapDataDictionary({
    dataDictionary: {
        "root": {
            schemaId: "foo",
            data: {
                greeting: "Hello world"
            }
        }
    },
    dataDictionaryKey: "root",
    schemaDictionary: {
        foo: {
            id: "foo",
            type: "object"
        }
    },
    mapper: mappingFunction
});

```

The expected result:

```javascript
{
    greeting: "Hello world"
}
```