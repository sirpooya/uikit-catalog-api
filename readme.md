# Catalog API for UIKit Catalog

A **catalog** is a single JSON file that tells the [UIKit Catalog](https://www.figma.com/community) Figma plugin which components to show, where their thumbnails are, and which Figma component to insert.

Point the plugin at any URL that returns this JSON. To try it instantly, use the sample in this repo:

```
https://raw.githubusercontent.com/sirpooya/catalog-public-api/refs/heads/main/sample-catalog.json
```

See [`sample-catalog.json`](./sample-catalog.json) for a complete, working example.

## JSON shape

```jsonc
{
  "libraries": [
    {
      "slug": "material-3-design-kit",    // unique id for the library
      "enum": "material-3-design-kit",     // value each component's `library` field uses
      "label": "Material 3 Design Kit",    // shown as a tab in the plugin
      "fileKey": "gHcBI1shhjMagRuv0ROmFD", // the Figma file the library lives in
      "count": 232                         // optional: number of components
    }
  ],
  "components": [
    {
      "library": "material-3-design-kit",   // must match a library `enum` (or `slug`)
      "libraryLabel": "Material 3 Design Kit",
      "name": "account_circle",             // shown on the card
      "key": "bfeba550011869f4352b4b7d…",   // Figma component key — REQUIRED to insert
      "nodeId": "54616-25459",              // optional: used by "Go to main component"
      "thumbnailUrl": "https://…/thumb.png",// optional: card preview image
      "description": "",                    // optional: shown in the detail drawer
      "width": null,                        // optional: natural px size (smart thumbnails)
      "height": null
    }
  ],
  "schemes": [                              // optional: mapping schemes, see below
    {
      "oldKey": "97205b2e01860d63…",        // deprecated component's key — the primary match
      "baseName": "Stepper Item",           // optional: fallback match if the key changed
      "library": "material-3-design-kit",   // optional: scopes the baseName fallback
      "url": "https://…/api/figma/scheme/97205b2e01860d63…"  // the scheme JSON
    }
  ]
}
```

### Required fields

| Field | Where | Why |
|-------|-------|-----|
| `libraries[].enum` (or `slug`) | each library | groups components into tabs |
| `libraries[].label` | each library | the tab name |
| `components[].name` | each component | the card label |
| `components[].key` | each component | **the Figma component key** — without it the component can't be inserted |
| `components[].library` | each component | links the component to a library `enum`/`slug` |

Everything else (`thumbnailUrl`, `description`, `width`, `height`, `nodeId`, `count`, `schemes`) is optional — the plugin degrades gracefully when a field is missing.

## `schemes[]` — migrating off a deprecated component

A **mapping scheme** is a JSON document describing how to migrate instances of a *deprecated* component onto its replacement. When the plugin finds a deprecated instance on the canvas, it looks the component up in `schemes[]`, fetches `url`, and uses the scheme to drive the migration instead of asking the designer to remap every property by hand.

`schemes[]` is a **top-level sibling** of `libraries[]` and `components[]` — never a field on a component entry:

```jsonc
{
  "libraries": [ /* … */ ],
  "components": [ /* … */ ],
  "schemes": [
    {
      "oldKey": "97205b2e01860d63ed3466c741711cebb8847dd6",
      "baseName": "Vertical Segmented Stepper - Stepper Item",
      "library": "core",
      "url": "https://dds.alamut.digikala.com/api/figma/scheme/97205b2e01860d63ed3466c741711cebb8847dd6"
    }
  ]
}
```

**Why top-level?** If your catalog is generated from the Figma REST API, the generator rebuilds `components[]` from scratch on every run — so any custom field you add to a component entry is wiped by the next sync. A separate top-level key survives it.

### How to match

1. **`oldKey` first.** It's the deprecated component's Figma key, and it's the reliable match.
2. **Fall back to `baseName` + `library`.** A republished component gets a *new* key, at which point the key match silently stops working. `baseName` is the component's name **without** the deprecation marker — i.e. what the live replacement is called. Always scope it by `library`: the same base name legitimately exists in more than one library, so an unscoped name match can land on the wrong component.

`baseName` and `library` are omitted when unknown rather than sent as `null`. `url` returns the raw scheme JSON with `Content-Type: application/json`, or `404` if none exists.

A consumer that ignores `schemes[]` keeps working exactly as before.

See the `schemes` entry in [`sample-catalog.json`](./sample-catalog.json) for a real one.

## How to get the `key` (Figma component key)

The `key` is the **published component key** Figma uses to import a component from a library. Read it via the [Figma REST API](https://www.figma.com/developers/api): `GET /v1/files/:file_key/components` returns each published component with its `key`, `name`, and `node_id`. Build your catalog by listing your library's components and copying those values.

> Inserting only works when the component's **library is published** and **enabled** in the user's file. Thumbnails (plain images) always show; the insert call fails with a permission error otherwise.

## Hosting

The plugin fetches the URL directly from its iframe, so your endpoint must:

1. Return the JSON above.
2. Send the header **`Access-Control-Allow-Origin: *`** (plugin iframes have a `null` origin, so CORS must be open).

A static file works fine — a GitHub **raw** URL (like this repo), a gist, or any static host. Dynamic APIs work too, as long as they set the CORS header.

## A note on thumbnails

Figma's image URLs (`s3-alpha.figma.com/…`) are **signed and expire** (typically a few days). If thumbnails stop loading, regenerate the catalog so the URLs are fresh, or host the images somewhere permanent and use those URLs instead.
