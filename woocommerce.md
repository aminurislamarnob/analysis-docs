# WooCommerce Variable Product & Variation Feature — Developer Documentation

> **Analysis of**: WooCommerce plugin  
> **Focus**: How the Variable Product / Variation feature is implemented  
> **Generated**: February 24, 2026

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Data Model](#2-data-model)
   - [Class Hierarchy](#21-class-hierarchy)
   - [Database Tables & Storage](#22-database-tables--storage)
   - [Product Attributes](#23-product-attributes)
   - [Data Stores](#24-data-stores)
3. [Admin UI Implementation](#3-admin-ui-implementation)
   - [Meta Boxes & Views](#31-meta-boxes--views)
   - [AJAX Handlers](#32-ajax-handlers)
   - [Admin JavaScript](#33-admin-javascript)
   - [Admin Workflow](#34-admin-workflow)
4. [Frontend Implementation](#4-frontend-implementation)
   - [Templates](#41-templates)
   - [Frontend JavaScript](#42-frontend-javascript)
   - [Frontend PHP Functions](#43-frontend-php-functions)
   - [Customer Flow](#44-customer-flow)
5. [REST API](#5-rest-api)
6. [Cart & Checkout Handling](#6-cart--checkout-handling)
7. [Key Hooks & Filters](#7-key-hooks--filters)
8. [Caching Strategy](#8-caching-strategy)
9. [Extending Variations (For Plugin Developers)](#9-extending-variations-for-plugin-developers)

---

## 1. Architecture Overview

WooCommerce's variable product system allows a single product to have multiple purchasable variations, each with its own price, stock, SKU, image, and other properties. The architecture is built on top of WordPress's Custom Post Type (CPT) system.

**Key architectural decisions:**

- **Variable Product** = a WooCommerce product (`post_type = 'product'`) with `product_type` taxonomy term = `'variable'`
- **Variation** = a child post (`post_type = 'product_variation'`) linked via `post_parent`
- **No custom tables** for variations — uses standard `wp_posts` + `wp_postmeta`
- **Attributes** drive variations — each variation is a unique combination of attribute values
- **Inheritance** — variations inherit parent data (title, SKU, weight, etc.) when their own values are empty
- **Caching** via transients for price aggregation and children IDs

```
┌──────────────────────────────────────┐
│  WC_Product_Variable                 │
│  post_type = 'product'               │
│  product_type taxonomy = 'variable'  │
│                                      │
│  _product_attributes (serialized)    │
│    ├── pa_color (is_variation=1)     │
│    └── pa_size  (is_variation=1)     │
│                                      │
│  Children (via post_parent):         │
│  ┌─────────────────────────────────┐ │
│  │ WC_Product_Variation #1         │ │
│  │ post_type = 'product_variation' │ │
│  │ attribute_pa_color = 'red'      │ │
│  │ attribute_pa_size  = 'large'    │ │
│  │ _price = 29.99                  │ │
│  ├─────────────────────────────────┤ │
│  │ WC_Product_Variation #2         │ │
│  │ attribute_pa_color = 'blue'     │ │
│  │ attribute_pa_size  = 'small'    │ │
│  │ _price = 24.99                  │ │
│  └─────────────────────────────────┘ │
└──────────────────────────────────────┘
```

---

## 2. Data Model

### 2.1 Class Hierarchy

```
WC_Data
  └── WC_Product (abstract-ish base)
        ├── WC_Product_Variable
        │     (post_type: 'product', type: 'variable')
        │
        └── WC_Product_Simple
              └── WC_Product_Variation
                    (post_type: 'product_variation')
```

#### `WC_Product_Variable` — `includes/class-wc-product-variable.php`

| Property | Type | Description |
|---|---|---|
| `$children` | `array` | All variation IDs (lazy-loaded) |
| `$visible_children` | `array` | Visible variation IDs |
| `$variation_attributes` | `array` | Attributes flagged for variations |

| Key Method | Description |
|---|---|
| `get_children()` | Returns all variation IDs via data store `read_children()` |
| `get_visible_children()` | Returns visible variation IDs (respects stock/status) |
| `get_variation_attributes()` | Returns attributes with `is_variation = 1` |
| `get_variation_prices()` | Aggregates min/max prices from all variations |
| `get_available_variations()` | Returns formatted variation data arrays for frontend |
| `get_available_variation($variation)` | Returns single variation's data (price, image, attributes, etc.) |

#### `WC_Product_Variation` — `includes/class-wc-product-variation.php`

| Property | Type | Description |
|---|---|---|
| `$parent_data` | `array` | Cached parent product data (title, SKU, stock settings, etc.) |

| Key Method | Description |
|---|---|
| `get_variation_attributes()` | Returns attributes with `attribute_` prefix |
| `get_parent_id()` | Returns `post_parent` from `wp_posts` |
| Inherited getters | Falls back to parent data when variation property is empty |

### 2.2 Database Tables & Storage

#### `wp_posts` table

| Column | Variable Product | Variation |
|---|---|---|
| `ID` | Product ID | Variation ID |
| `post_type` | `'product'` | `'product_variation'` |
| `post_parent` | `0` | `{parent_product_id}` |
| `post_status` | `'publish'` | `'publish'` |

#### `wp_postmeta` table

**Variable Product meta keys:**

| Meta Key | Value | Description |
|---|---|---|
| `_product_attributes` | Serialized array | All product attributes (see 2.3) |
| `_price` | Multiple entries | One per unique child price (for query optimization) |

**Variation meta keys:**

| Meta Key | Example Value | Description |
|---|---|---|
| `attribute_pa_color` | `'blue'` | Taxonomy attribute value (term slug) |
| `attribute_size` | `'Large'` | Custom attribute value (text) |
| `_regular_price` | `'29.99'` | Regular price |
| `_sale_price` | `'24.99'` | Sale price |
| `_price` | `'24.99'` | Active price |
| `_stock_status` | `'instock'` | Stock status |
| `_stock` | `'50'` | Stock quantity |
| `_manage_stock` | `'yes'` | Whether to manage stock |
| `_sku` | `'PROD-VAR-001'` | SKU (inherits parent if empty) |
| `_weight`, `_length`, `_width`, `_height` | Numeric | Shipping dimensions |
| `_tax_class` | `'parent'` | Tax class (or `'parent'` to inherit) |
| `_variation_description` | Text | Variation-specific description |
| `_thumbnail_id` | Attachment ID | Variation image |

#### `wc_product_meta_lookup` table (performance)

Both variable products and their variations have entries in this lookup table for fast queries:

```sql
CREATE TABLE wc_product_meta_lookup (
    product_id      BIGINT(20) NOT NULL,
    sku             VARCHAR(100),
    virtual         TINYINT(1) DEFAULT 0,
    downloadable    TINYINT(1) DEFAULT 0,
    min_price       DECIMAL(19,4),
    max_price       DECIMAL(19,4),
    onsale          TINYINT(1) DEFAULT 0,
    stock_quantity  DOUBLE,
    stock_status    VARCHAR(100) DEFAULT 'instock',
    rating_count    BIGINT(20) DEFAULT 0,
    average_rating  DECIMAL(3,2) DEFAULT 0.00,
    total_sales     BIGINT(20) DEFAULT 0,
    tax_status      VARCHAR(100) DEFAULT 'taxable',
    tax_class       VARCHAR(100) DEFAULT '',
    PRIMARY KEY (product_id)
);
```

#### `wp_term_relationships` & `wp_term_taxonomy`

Used for taxonomy-based attributes (e.g., `pa_color`, `pa_size`). Links products to attribute terms via `wp_set_object_terms()`.

### 2.3 Product Attributes

**File:** `includes/class-wc-product-attribute.php`

Each attribute is stored in the parent product's `_product_attributes` meta as a serialized array:

```php
// Structure of _product_attributes meta value
[
    'pa_color' => [
        'name'         => 'pa_color',        // Taxonomy slug or custom name
        'value'        => '',                 // Empty for taxonomy attributes
        'position'     => 0,
        'is_visible'   => 1,                 // Show in "Additional Information" tab
        'is_variation' => 1,                 // KEY FLAG: Used for variations
        'is_taxonomy'  => 1,                 // 1 = taxonomy, 0 = custom
    ],
    'custom-attr' => [
        'name'         => 'Custom Attribute',
        'value'        => 'Option 1|Option 2', // Pipe-separated for custom
        'position'     => 1,
        'is_visible'   => 0,
        'is_variation' => 1,
        'is_taxonomy'  => 0,
    ],
]
```

**Two types of attributes:**

| Type | `is_taxonomy` | Storage | Options Format |
|---|---|---|---|
| **Taxonomy-based** | `1` | WordPress taxonomy terms (`pa_*`) | Term IDs in `wp_term_relationships` |
| **Custom/Text** | `0` | Serialized in `_product_attributes` | Pipe-separated string (`"S\|M\|L\|XL"`) |

### 2.4 Data Stores

#### `WC_Product_Variable_Data_Store_CPT`

**File:** `includes/data-stores/class-wc-product-variable-data-store-cpt.php`  
**Extends:** `WC_Product_Data_Store_CPT`

| Method | Description |
|---|---|
| `read_children()` | Queries `wp_posts` where `post_parent = {id}` and `post_type = 'product_variation'`. Cached in transient `wc_product_children_{id}` (30 days). |
| `read_variation_attributes()` | Builds attribute/value pairs from variation `attribute_*` meta keys |
| `read_price_data()` | Aggregates prices from all variations. Cached in transient `wc_var_prices_{id}`. |
| `sync_price()` | Syncs parent product meta `_price` with child variation prices |
| `sync_stock_status()` | Sets parent stock status based on children availability |

#### `WC_Product_Variation_Data_Store_CPT`

**File:** `includes/data-stores/class-wc-product-variation-data-store-cpt.php`  
**Extends:** `WC_Product_Data_Store_CPT`

| Method | Description |
|---|---|
| `create()` | Inserts `wp_posts` row with `post_type = 'product_variation'` and `post_parent` |
| `read()` | Loads from `wp_posts`, validates `post_type = 'product_variation'` |
| `update_attributes()` | Saves attributes as `attribute_{key}` meta keys |
| `read_product_data()` | Loads variation-specific meta + parent data for inheritance |
| `generate_product_title()` | Creates title: `"{Parent Title} - {Attribute1}, {Attribute2}"` |

---

## 3. Admin UI Implementation

### 3.1 Meta Boxes & Views

#### Main Meta Box

**File:** `includes/admin/meta-boxes/class-wc-meta-box-product-data.php`  
**Class:** `WC_Meta_Box_Product_Data`

| Method | Description |
|---|---|
| `output()` | Renders the main product data metabox with tabs |
| `output_variations()` | Renders the variations tab content |
| `save_variations()` | Saves variation data from POST |
| `prepare_attributes()` | Prepares attributes for saving |

#### Admin Views (HTML Templates)

| File | Purpose |
|---|---|
| `includes/admin/meta-boxes/views/html-product-data-panel.php` | Main product data panel with tabs |
| `includes/admin/meta-boxes/views/html-product-data-attributes.php` | Attributes tab (add/edit attributes, "Used for variations" checkbox) |
| `includes/admin/meta-boxes/views/html-product-data-variations.php` | Variations tab (generate/add buttons, bulk actions, pagination, Backbone container) |
| `includes/admin/meta-boxes/views/html-variation-admin.php` | Individual variation form (SKU, pricing, inventory, shipping, image, description) |

**The Variations tab** (`#variable_product_options`) includes:
- Default form values selector
- "Generate variations" button — creates all attribute combinations
- "Add manually" button — adds a single blank variation
- Bulk actions dropdown (set prices, stock, toggle status, etc.)
- Pagination controls
- `.woocommerce_variations` container — Backbone.js managed

### 3.2 AJAX Handlers

**File:** `includes/class-wc-ajax.php` — `WC_AJAX` class

| AJAX Action | Method | Nonce | Description |
|---|---|---|---|
| `woocommerce_load_variations` | `load_variations()` | `load-variations` | Loads paginated variations |
| `woocommerce_save_variations` | `save_variations()` | `save-variations` | Saves variation data |
| `woocommerce_add_variation` | `add_variation()` | `add-variation` | Creates a single new variation |
| `woocommerce_remove_variations` | `remove_variations()` | `delete-variations` | Deletes selected variations |
| `woocommerce_add_attributes_and_variations` | `add_attributes_and_variations()` | `add-attributes-and-variations` | Creates attributes + generates all variation combinations |
| `woocommerce_bulk_edit_variations` | `bulk_edit_variations()` | `bulk-edit-variations` | Bulk operations (pricing, stock, status, etc.) |

### 3.3 Admin JavaScript

**File:** `assets/js/admin/meta-boxes-product-variation.js`

| JS Object | Responsibility |
|---|---|
| `wc_meta_boxes_product_variations_actions` | UI interactions: create variations, manage downloadable/virtual/stock toggles, menu ordering |
| `wc_meta_boxes_product_variations_ajax` | AJAX calls: `load_variations()`, `save_variations()`, `save_changes()`, `add_variation_manually()`, `generate_variations()`, `remove_variation()` |
| `wc_meta_boxes_product_variations_media` | Variation image uploads via WordPress media library |
| `wc_meta_boxes_product_variations_pagenav` | Pagination controls for variation list |

**Key UI behaviors:**
- Uses **Backbone.js** for variation management
- **jQuery** for DOM manipulation
- Change tracking via `.variation-needs-update` CSS class
- **jQuery UI Sortable** for reordering variations
- **Date pickers** for sale price scheduling
- Only changed variations are sent on save (optimized AJAX payload)

### 3.4 Admin Workflow

```
┌─ Step 1: Product Type ────────────────────────────────┐
│  User selects "Variable product" from Product Data     │
│  dropdown. This triggers show/hide of relevant tabs.   │
└────────────────────────────────────────────────────────┘
         │
         ▼
┌─ Step 2: Attributes Tab ──────────────────────────────┐
│  User adds attributes (taxonomy or custom)             │
│  ☑ "Used for variations" checkbox per attribute         │
│  Saves attributes to _product_attributes meta          │
└────────────────────────────────────────────────────────┘
         │
         ▼
┌─ Step 3: Variations Tab ──────────────────────────────┐
│  Option A: "Generate variations" button                │
│    → AJAX: woocommerce_add_attributes_and_variations   │
│    → Creates all possible combinations                 │
│                                                        │
│  Option B: "Add manually" button                       │
│    → AJAX: woocommerce_add_variation                   │
│    → Creates one blank variation                       │
└────────────────────────────────────────────────────────┘
         │
         ▼
┌─ Step 4: Edit Variations ─────────────────────────────┐
│  Each variation loaded via AJAX pagination              │
│  User sets: price, SKU, stock, image, dimensions, etc. │
│  Changes tracked with .variation-needs-update class     │
└────────────────────────────────────────────────────────┘
         │
         ▼
┌─ Step 5: Save ────────────────────────────────────────┐
│  "Save changes" button → AJAX: woocommerce_save_       │
│  variations → WC_Meta_Box_Product_Data::save_           │
│  variations() → Each variation saved via                │
│  WC_Product_Variation->save()                          │
└────────────────────────────────────────────────────────┘
```

---

## 4. Frontend Implementation

### 4.1 Templates

| Template File | Purpose |
|---|---|
| `templates/single-product/add-to-cart/variable.php` | Main variable product form: renders attribute dropdowns, embeds variation data in `data-product_variations` attribute |
| `templates/single-product/add-to-cart/variation.php` | JS templates: `tmpl-variation-template` (displays price, description, availability) and `tmpl-unavailable-variation-template` |
| `templates/single-product/add-to-cart/variation-add-to-cart-button.php` | Quantity input + Add to Cart button + hidden `variation_id` field |

**Template loading chain:**

```
woocommerce_single_product_summary hook (priority 30)
  └── woocommerce_template_single_add_to_cart()
        └── woocommerce_variable_add_to_cart()  [wc-template-functions.php]
              ├── Enqueues 'wc-add-to-cart-variation' script
              ├── Loads variable.php template
              └── Threshold check: variations ≤ 30 → inline JSON
                                   variations > 30 → AJAX mode
```

**Inline vs AJAX mode:**

The filter `woocommerce_ajax_variation_threshold` (default: `30`) controls whether variation data is embedded inline in the HTML or loaded via AJAX:

- **Inline mode**: All variation data as JSON in `data-product_variations` attribute on the `<form>`
- **AJAX mode**: `data-product_variations="false"`, each variation fetched on demand

### 4.2 Frontend JavaScript

**File:** `assets/js/frontend/add-to-cart-variation.js`

**Core class:** `VariationForm` — initialized as jQuery plugin `$.fn.wc_variation_form()`

| Method | Description |
|---|---|
| `getChosenAttributes()` | Collects all selected attribute values from dropdowns |
| `findMatchingVariations()` | Matches selected attributes against variation data array |
| `onChange()` | Handles dropdown `change` event, triggers variation check |
| `onFindVariation()` | Finds matching variation (inline match or AJAX request) |
| `onFoundVariation()` | Updates UI: price, image, availability, SKU, dimensions, button state |
| `onUpdateAttributes()` | Filters dropdown options based on other selected attributes (cascading) |

**Localized parameters:**
- Object: `wc_add_to_cart_variation_params`
- Set in: `includes/class-wc-frontend-scripts.php`
- Contains: `wc_ajax_url`, i18n strings

**Image update function:** `wc_variations_image_update()` — swaps main product image and gallery thumbnail when a variation with its own image is selected.

### 4.3 Frontend PHP Functions

**File:** `includes/wc-template-functions.php`

| Function | Description |
|---|---|
| `wc_dropdown_variation_attribute_options()` | Renders `<select>` dropdown for an attribute. Sets `data-attribute_name` and `data-show_option_none`. Handles both taxonomy and custom attributes. |
| `woocommerce_single_variation()` | Outputs empty `<div class="single_variation">` placeholder populated by JS |
| `woocommerce_single_variation_add_to_cart_button()` | Loads the add-to-cart button template |

**Key method on `WC_Product_Variable`:**

`get_available_variations()` returns an array of variation data arrays, each containing:

```php
[
    'variation_id'           => 123,
    'variation_is_visible'   => true,
    'variation_is_active'    => true,
    'is_purchasable'         => true,
    'display_price'          => 29.99,
    'display_regular_price'  => 34.99,
    'attributes'             => ['attribute_pa_color' => 'blue', 'attribute_pa_size' => 'large'],
    'price_html'             => '<span class="price">$29.99</span>',
    'availability_html'      => '<p class="stock in-stock">In stock</p>',
    'image'                  => [...],  // Full image data array
    'sku'                    => 'PROD-001-BLU-L',
    'weight'                 => '0.5',
    'dimensions'             => ['length' => '10', 'width' => '5', 'height' => '2'],
    'variation_description'  => 'Blue large variant description',
    'is_in_stock'            => true,
    'max_qty'                => 50,
    'min_qty'                => 1,
    // ... more fields
]
```

### 4.4 Customer Flow

```
┌─ Page Load ───────────────────────────────────────────┐
│ 1. Template loads: variable.php                        │
│ 2. Attribute dropdowns rendered via                    │
│    wc_dropdown_variation_attribute_options()            │
│ 3. Variation data embedded (inline) or AJAX flag set   │
│ 4. JS initializes VariationForm on .variations_form    │
└────────────────────────────────────────────────────────┘
         │
         ▼
┌─ Attribute Selection ─────────────────────────────────┐
│ 1. User selects attribute (e.g., Color = Blue)         │
│ 2. change event → onChange()                           │
│ 3. Clears variation_id hidden field                    │
│ 4. Triggers check_variations event                     │
│ 5. Other dropdowns filtered (onUpdateAttributes)       │
└────────────────────────────────────────────────────────┘
         │
         ▼
┌─ Variation Matching ──────────────────────────────────┐
│ All attributes selected?                               │
│   ├── YES + Inline mode:                               │
│   │     findMatchingVariations() against variationData │
│   ├── YES + AJAX mode:                                 │
│   │     POST to wc_ajax get_variation endpoint         │
│   └── NO:                                              │
│         Show "select options" state                    │
│                                                        │
│ Match found → found_variation event                    │
│ No match   → show_variation event (unavailable)        │
└────────────────────────────────────────────────────────┘
         │
         ▼
┌─ UI Update ───────────────────────────────────────────┐
│ onFoundVariation() updates:                            │
│   • Price HTML (via JS template)                       │
│   • Product image (wc_variations_image_update)         │
│   • SKU, weight, dimensions                            │
│   • Stock availability                                 │
│   • Hidden variation_id field                          │
│   • Add to cart button enabled/disabled                │
└────────────────────────────────────────────────────────┘
         │
         ▼
┌─ Add to Cart ─────────────────────────────────────────┐
│ 1. Form submits: variation_id + all attribute_* fields │
│ 2. add_to_cart_handler_variable() [WC_Form_Handler]    │
│ 3. Validates variation_id belongs to product           │
│ 4. Validates all required attributes provided          │
│ 5. WC_Cart::add_to_cart() with variation data          │
│ 6. Cart key = hash(product_id, variation_id, attrs)    │
└────────────────────────────────────────────────────────┘
```

---

## 5. REST API

**File:** `includes/rest-api/Controllers/Version3/class-wc-rest-product-variations-controller.php`

**Namespace:** `wc/v3`  
**Base route:** `/products/{product_id}/variations`

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/wp-json/wc/v3/products/{id}/variations` | List all variations |
| `GET` | `/wp-json/wc/v3/products/{id}/variations/{vid}` | Get single variation |
| `POST` | `/wp-json/wc/v3/products/{id}/variations` | Create variation |
| `PUT` | `/wp-json/wc/v3/products/{id}/variations/{vid}` | Update variation |
| `DELETE` | `/wp-json/wc/v3/products/{id}/variations/{vid}` | Delete variation |
| `POST` | `/wp-json/wc/v3/products/{id}/variations/generate` | Generate all variation combinations |

**Frontend AJAX endpoint:**

| Endpoint | File | Method |
|---|---|---|
| `WC_AJAX::get_endpoint('get_variation')` | `includes/class-wc-ajax.php` | `get_variation()` |

Process:
1. Receives `product_id` + attribute values via POST
2. Uses `WC_Data_Store::find_matching_product_variation()` to locate matching variation
3. Returns `get_available_variation()` data as JSON

---

## 6. Cart & Checkout Handling

### Adding to Cart

**File:** `includes/class-wc-form-handler.php`  
**Method:** `add_to_cart_handler_variable()`

1. Extracts `variation_id` and `quantity` from `$_POST`
2. Collects all `attribute_*` fields into `$variations` array
3. Validates via `woocommerce_add_to_cart_validation` filter
4. Calls `WC()->cart->add_to_cart()` with variation data

**File:** `includes/class-wc-cart.php`  
**Method:** `add_to_cart()`

For variations specifically:
1. Validates `variation_id` belongs to `product_id` (parent check)
2. Validates all required attributes are provided
3. Finds matching variation if `variation_id` not explicitly provided
4. Generates unique cart item key from: `product_id + variation_id + variation_attributes`
5. Stores cart item with both product and variation references

### Cart Item Data Structure

```php
$cart_item = [
    'key'               => 'abc123...',          // MD5 hash
    'product_id'        => 100,                   // Parent variable product
    'variation_id'      => 105,                   // Specific variation
    'variation'         => [                      // Selected attributes
        'attribute_pa_color' => 'blue',
        'attribute_pa_size'  => 'large',
    ],
    'quantity'          => 2,
    'data'              => WC_Product_Variation,  // Product object
    'line_tax_data'     => [...],
    'line_subtotal'     => 59.98,
    'line_subtotal_tax' => 0,
    'line_total'        => 59.98,
    'line_tax'          => 0,
];
```

---

## 7. Key Hooks & Filters

### Action Hooks

| Hook | Location | Description |
|---|---|---|
| `woocommerce_variable_product_before_variations` | Variations panel | Before the variations list |
| `woocommerce_variation_header` | Variation metabox | In variation header |
| `woocommerce_variation_options` | Variation metabox | Variation option checkboxes (enabled, downloadable, virtual) |
| `woocommerce_variation_options_pricing` | Variation metabox | After pricing fields |
| `woocommerce_variation_options_inventory` | Variation metabox | After inventory fields |
| `woocommerce_variation_options_dimensions` | Variation metabox | After dimensions fields |
| `woocommerce_save_product_variation` | Save process | After a variation is saved (receives `$variation_id`, `$loop_index`) |
| `woocommerce_admin_process_variation_object` | Save process | Before a variation object is saved (receives `$variation`, `$loop_index`) |
| `woocommerce_new_product_variation` | Creation | When a new variation post is created |
| `woocommerce_update_product_variation` | Update | When a variation post is updated |

### Filter Hooks

| Hook | Default | Description |
|---|---|---|
| `woocommerce_ajax_variation_threshold` | `30` | Max variations for inline data (above = AJAX mode) |
| `woocommerce_available_variation` | — | Modify single variation data array for frontend |
| `woocommerce_show_variation_price` | — | Control when to show variation price |
| `woocommerce_hide_invisible_variations` | `true` | Whether to hide out-of-stock/disabled variations |
| `product_type_selector` | — | Modify available product types |
| `woocommerce_product_data_tabs` | — | Modify product data tabs |
| `woocommerce_dropdown_variation_attribute_options_args` | — | Modify dropdown rendering arguments |
| `woocommerce_dropdown_variation_attribute_options_html` | — | Modify dropdown HTML output |
| `woocommerce_admin_meta_boxes_variations_per_page` | `15` | Variations per page in admin |
| `woocommerce_add_to_cart_validation` | — | Validate before adding variation to cart |

---

## 8. Caching Strategy

WooCommerce uses transients to cache expensive variation queries:

| Transient Key | TTL | Content |
|---|---|---|
| `wc_product_children_{product_id}` | 30 days | Array of all variation IDs (visible + all) |
| `wc_var_prices_{product_id}` | 30 days | Aggregated price data (min/max regular, sale, active prices) |

**Cache invalidation** occurs when:
- A variation is created, updated, or deleted
- Parent product is saved
- `wc_delete_product_transients()` is called
- WooCommerce status tools "Clear transients" is run

---

## 9. Extending Variations (For Plugin Developers)

### Adding Custom Fields to Variations

```php
// 1. Add field to admin variation form
add_action( 'woocommerce_variation_options_pricing', function( $loop, $variation_data, $variation ) {
    woocommerce_wp_text_input( [
        'id'          => "custom_field_{$loop}",
        'name'        => "custom_field[{$loop}]",
        'label'       => __( 'Custom Field', 'your-plugin' ),
        'value'       => get_post_meta( $variation->ID, '_custom_field', true ),
        'wrapper_class' => 'form-row form-row-first',
    ] );
}, 10, 3 );

// 2. Save the custom field
add_action( 'woocommerce_save_product_variation', function( $variation_id, $loop ) {
    if ( isset( $_POST['custom_field'][ $loop ] ) ) {
        update_post_meta(
            $variation_id,
            '_custom_field',
            sanitize_text_field( wp_unslash( $_POST['custom_field'][ $loop ] ) )
        );
    }
}, 10, 2 );

// 3. Add data to frontend variation JSON
add_filter( 'woocommerce_available_variation', function( $data, $product, $variation ) {
    $data['custom_field'] = $variation->get_meta( '_custom_field' );
    return $data;
}, 10, 3 );
```

### Modifying Variation Dropdowns

```php
// Change dropdown HTML
add_filter( 'woocommerce_dropdown_variation_attribute_options_html', function( $html, $args ) {
    // Convert select to radio buttons, add swatches, etc.
    return $html;
}, 10, 2 );
```

### Adding Custom Product Type with Variations

```php
// Register as variable type
add_filter( 'product_type_selector', function( $types ) {
    $types['custom_variable'] = __( 'Custom Variable Product', 'your-plugin' );
    return $types;
} );
```

### Programmatically Creating a Variable Product with Variations

```php
// Create variable product
$product = new WC_Product_Variable();
$product->set_name( 'Test Variable Product' );
$product->set_status( 'publish' );

// Set attributes
$attribute = new WC_Product_Attribute();
$attribute->set_name( 'pa_color' );
$attribute->set_options( [ 'red', 'blue', 'green' ] );
$attribute->set_visible( true );
$attribute->set_variation( true );
$product->set_attributes( [ $attribute ] );
$product->save();

// Create variations
foreach ( [ 'red', 'blue', 'green' ] as $color ) {
    $variation = new WC_Product_Variation();
    $variation->set_parent_id( $product->get_id() );
    $variation->set_attributes( [ 'pa_color' => $color ] );
    $variation->set_regular_price( 29.99 );
    $variation->set_stock_status( 'instock' );
    $variation->save();
}
```

---

## File Reference Index

| Category | File Path |
|---|---|
| **Data Model** | |
| Variable Product Class | `includes/class-wc-product-variable.php` |
| Variation Class | `includes/class-wc-product-variation.php` |
| Product Attribute Class | `includes/class-wc-product-attribute.php` |
| Variable Data Store | `includes/data-stores/class-wc-product-variable-data-store-cpt.php` |
| Variation Data Store | `includes/data-stores/class-wc-product-variation-data-store-cpt.php` |
| Product Factory | `includes/class-wc-product-factory.php` |
| Product Type Enum | `src/Enums/ProductType.php` |
| **Admin** | |
| Product Meta Box | `includes/admin/meta-boxes/class-wc-meta-box-product-data.php` |
| Attributes Tab View | `includes/admin/meta-boxes/views/html-product-data-attributes.php` |
| Variations Tab View | `includes/admin/meta-boxes/views/html-product-data-variations.php` |
| Variation Form View | `includes/admin/meta-boxes/views/html-variation-admin.php` |
| Admin JS (Variations) | `assets/js/admin/meta-boxes-product-variation.js` |
| AJAX Handler | `includes/class-wc-ajax.php` |
| **Frontend** | |
| Variable Add-to-Cart Template | `templates/single-product/add-to-cart/variable.php` |
| Variation JS Template | `templates/single-product/add-to-cart/variation.php` |
| Add-to-Cart Button Template | `templates/single-product/add-to-cart/variation-add-to-cart-button.php` |
| Frontend JS | `assets/js/frontend/add-to-cart-variation.js` |
| Template Functions | `includes/wc-template-functions.php` |
| Frontend Scripts | `includes/class-wc-frontend-scripts.php` |
| **Cart** | |
| Cart Class | `includes/class-wc-cart.php` |
| Form Handler | `includes/class-wc-form-handler.php` |
| **REST API** | |
| Variations Controller | `includes/rest-api/Controllers/Version3/class-wc-rest-product-variations-controller.php` |
| **Registration** | |
| Post Types | `includes/class-wc-post-types.php` |
| Product Functions | `includes/wc-product-functions.php` |

---

> **Note:** All file paths are relative to the WooCommerce plugin root: `wp-content/plugins/woocommerce/`
