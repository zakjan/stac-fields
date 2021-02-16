# stac-fields

A minimal STAC library that contains a list of STAC fields with some metadata (title, unit, prefix) and helper functions.

Version: **1.0.0-beta.1**

## Usage

Add to your project with `npm install @radiantearth/stac-fields --save`

Import the utilities to format values:
`const StacFields = require('@radiantearth/stac-fields');`

Format a value:
```js
let stacItem = {
    stac_version: '1.0.0',
    id: '...',
    properties: {
        datetime: '2020-01-01T13:55:43Z',
        'radiant:public_access': true,
        ...
    },
    ...
};

// Add custom extension and field(s)
StacFields.Registry.addExtension('radiant', 'Radiant Earth');
StacFields.Registry.addMetadataField('radiant:public_access', {
    label: "Data Access",
    formatter: (value) => value ? "Public" : "Private"
});

// Option 1: Manually iterate through properties and format them
for(let field in stacItem.properties) {
    let value = stacItem.properties[field];
    let formatted = StacFields.format(value, field, stacItem);
    let label = StacFields.label(field);
    console.log(label, formatted);
}

// Option 2: Group by extension and format item properties
// The second parameter is a filter function to skip specific properties, remove to get all properties
let groups = StacFields.formatStacProperties(stacItem, key => key !== 'eo:bands');
```

This library is written for the latest version of the STAC specification (1.0.0-beta.2).
It is recommended to pass your STAC data through a migration tool like `@radiantearth/stac-migrate` (WIP) before so that it complies to the latest STAC version. Otherwise some fields may not be handled correctly.

Non-JavaScript library authors can re-use the `fields.json`. It is available at:
<https://cdn.jsdelivr.net/npm/@radiantearth/stac-fields/fields.json>

## fields.json

The following options are available in the object:

* `label`: The human-readable title for the value.
* `format`: The name of the formatter in formatters.js, but without the leading `format`.
* `formatter`: A formatter function that is compatible to the formatters defined in formatters.js. Use this if no suitable pre-defined formatter is available to be specified in `format`.
* `unit`: A unit to add after the value.
* `explain`: A long form for an abbreviation that should be shown in a tooltip.
* `custom`: A structure that can't easily be rendered with any of the pre-defined formatters and thus needs a custom implementation (see `externalRenderer`).
* `alias`: If a field has multiple keys, declare the field to use the specification of the other field.
* `listWithKeys`: Set to `true` to allow the option `items` to specify items in an object of objects (like assets). Defaults to `false`.
* `items`: If the value is an array of objects (or an object of objects if `listWithKeys` has ben set), a table can be created with the details in this object. It has the same structure as specified here, but in additon `sortable` and `id` are allowed:
    * `sortable`: Specfiies whether the value can be sorted (`true`, e.g. in a table) or not (`false`). Defaults to `false`.
    * `id`: Specfiies whether the value is the unique primary key (`true`) or not (`false`). Defaults to `false`.
* `null`: The value that should be given instead of `null`. If a value is null but this property is not given, defaults to "n/a".
* Options related to Collection Summaries:
    * `summary`: If the fields should be added to summaries (`true`, default) or not `false`.
    * `mergeArrays`: If the values are arrays and should usually be merged in summaries (`true`, e.g. `eo:bands` or `instruments`) or not (`false`, default).
    * `order`: The order of the items in ascending order, e.g. for a table. If not given, the first entry is always the item with `id` set to `true`, all other items are in alphabetic order.

If only a label is available, it can be passed as string instead of an object.

Some details about the fields included in the fields.json file can be found [here](fields.md).

There is also a `fields-normalized.json`, which is a normalized version of the fields.json.
All non-Javascript users will probably prefer to use the `fields-normalized.json` as it already
has the `alias`es resolved and all fields and extensions are defined as objects for easier handling.

## formatters.js

The most important methods are:

* `format(value, field, spec, context = null) => string`: Applies the right formatting depending on the data type of the a single property.
* label
* extension
* formatSummaries
* formatItemProperties

### Pre-defined formatters (`Formatters`)

* CommonMark
* DOI
* EPSG
* Extent (array with two values)
* License
* Providers
* Software
* TemporalExtent (array with two timestamps)
* Timestamp (ISO8601 timestamp)
* Summary
* CSV (array to comma-separated values)
* FileSize
* FileDataType
* Checksum (multihashes)

### Custom formatters

Formatters are always functions that have the following signature:

`method(value : any, field : string, spec : object, context = null) => string`

- `value` is the value of the field in the STAC JSON
- `field` is the key of the field in the STAC JSON
- `spec` is the normalized object for the field from the `fields.json`.
- `context` is the full STAC JSON

The returned value is always expected to be a string.
It may contain HTML if the formatter is added to the `htmlFormats` array in `fields.json`.
If the return value is allowed to contain HTML, ALL user input must run thorugh the `e()` function (or `parseInt` for integers, for example) to escape malicious HTML tags.
This avoids XSS and similar security issues.

### `Registry`

* `externalRenderer` (boolean): Set to `true` to not render custom objects, arrays and tables with the renderers from this library. Will return formatted values individually then. Defaults to `false`.
* `addExtension(prefix : string, spec : object) => void` - Adds a additional (custom) extension that is compliant to the fields.json, can also be used to replace existing extensions
* `addMetadataField(field : string, spec : object) => void` - Adds a additional (custom) metdata field that is compliant to the fields.json, can also be used to replace existing fields
* `addMetadataFields(specs : object) => void` - Adds additional (custom) metdata fields that are compliant to the fields.json, can also be used to replace existing fields

### Data Types (`DataTypes`)

This object has functions to format the native JSON data types: 
* array => `array(arr, sort = false)`
* object => `object(obj)`
* null => `null(label = 'n/a')`
* number => `number(num)`
* string => `string(str)`
* boolean => `boolean(bool)`

Additionally, it has a method `format(value)`, which applies the right formatting depending on the data type of the value.

All methods return strings, which may contain HTML. Input is sanitized.

### `Helpers`

* e
* formatKey
* groupByExtensions
* isObject
* toLink
* toList
* toObject
* normalizeFields
* hextoUint8
* uint8ToHex