# Reusing WooCommerce PHP, AJAX & JS Functions for Variable Products — StoreSuite Guide

> **Goal**: Minimize custom code by leveraging WooCommerce's existing public functions, AJAX endpoints, and JavaScript assets when implementing variable product variations in StoreSuite's frontend product form.  
> **Generated**: February 24, 2026

---

## Table of Contents

1. [Reuse Strategy Summary](#1-reuse-strategy-summary)
2. [PHP Functions You Can Call Directly](#2-php-functions-you-can-call-directly)
   - [Attribute Functions](#21-attribute-functions)
   - [Product Functions](#22-product-functions)
   - [CRUD Classes (Create/Read/Update Variations)](#23-crud-classes)
   - [Data Store Methods](#24-data-store-methods)
   - [Template Functions](#25-template-functions)
3. [WooCommerce AJAX Endpoints You Can Reuse](#3-woocommerce-ajax-endpoints-you-can-reuse)
   - [Directly Reusable (No Custom AJAX Needed)](#31-directly-reusable)
   - [Admin-Only (Must Build Your Own)](#32-admin-only-must-build-your-own)
4. [JavaScript Assets You Can Enqueue](#4-javascript-assets-you-can-enqueue)
   - [Already Used by StoreSuite](#41-already-used-by-storesuite)
   - [Ready to Enqueue (Frontend-Safe)](#42-ready-to-enqueue)
   - [Admin-Only (Extract Logic Instead)](#43-admin-only-extract-logic)
5. [CSS You Can Reuse](#5-css-you-can-reuse)
6. [Practical Implementation: What to Reuse vs Build](#6-practical-implementation)
7. [Code Examples: Reusing WooCommerce Functions](#7-code-examples)
8. [What StoreSuite Already Reuses](#8-what-storesuite-already-reuses)

---

## 1. Reuse Strategy Summary

| Area | Reuse Level | What to Do |
|---|---|---|
| **PHP attribute functions** | **Full reuse** | Call WC functions directly — no custom code needed |
| **PHP CRUD classes** | **Full reuse** | Use `WC_Product_Variable`, `WC_Product_Variation`, `WC_Product_Attribute` directly |
| **PHP product sync** | **Full reuse** | Call `WC_Product_Variable::sync()` after saving variations |
| **AJAX `json_search_*`** | **Full reuse** | Already used by StoreSuite for upsells/cross-sells |
| **AJAX save/load/add/remove variations** | **Cannot reuse** | Admin-only (`edit_products` cap + admin DOM HTML) — build custom |
| **JS `selectWoo`** | **Already reused** | StoreSuite already enqueues it |
| **JS `wc-accounting`** | **Already reused** | StoreSuite already enqueues it |
| **JS admin variation scripts** | **Cannot reuse** | Tightly coupled to admin DOM — build custom |
| **JS `jquery-blockui`** | **Can reuse** | Enqueue for loading states |
| **JS `wc-enhanced-select`** | **Can reuse** | For attribute term search selects |
| **CSS** | **Partial reuse** | StoreSuite has its own design system — cherry-pick selectively |

---

## 2. PHP Functions You Can Call Directly

### 2.1 Attribute Functions

**File**: `woocommerce/includes/wc-attribute-functions.php`  
**All are global functions — call anywhere after WooCommerce loads.**

#### Get All Registered Attribute Taxonomies

```php
$attribute_taxonomies = wc_get_attribute_taxonomies();
// Returns: array of objects with ->attribute_id, ->attribute_name, ->attribute_label,
//          ->attribute_type, ->attribute_orderby, ->attribute_public
```

**Use in StoreSuite**: Populate the "Add Attribute" dropdown in the frontend form.

#### Get Attribute Taxonomy Name (with `pa_` prefix)

```php
$taxonomy_name = wc_attribute_taxonomy_name( 'color' );
// Returns: 'pa_color'
```

#### Get Attribute Label (Human-Readable)

```php
$label = wc_attribute_label( 'pa_color' );
// Returns: 'Color'

// For custom attributes on a specific product:
$label = wc_attribute_label( 'My Custom Attr', $product );
// Returns: 'My Custom Attr'
```

**Use in StoreSuite**: Display attribute names in the variation header and attribute rows.

#### Convert Between Name and ID

```php
$id   = wc_attribute_taxonomy_id_by_name( 'pa_color' );  // Returns: 5
$name = wc_attribute_taxonomy_name_by_id( 5 );            // Returns: 'pa_color'
```

**Use in StoreSuite**: When building `WC_Product_Attribute` objects, you need the ID for taxonomy attributes.

#### Parse Text Attributes

```php
$values = wc_get_text_attributes( 'Small | Medium | Large' );
// Returns: ['Small', 'Medium', 'Large']

$string = wc_implode_text_attributes( ['Small', 'Medium', 'Large'] );
// Returns: 'Small | Medium | Large'
```

**Use in StoreSuite**: Parse custom attribute pipe-separated values.

#### Get All Taxonomy Names and Labels

```php
$names  = wc_get_attribute_taxonomy_names();   // ['pa_color', 'pa_size', ...]
$labels = wc_get_attribute_taxonomy_labels();   // ['pa_color' => 'Color', ...]
$ids    = wc_get_attribute_taxonomy_ids();       // ['pa_color' => 5, ...]
```

#### Get Single Attribute Object

```php
$attribute = wc_get_attribute( 5 );
// Returns: stdClass { id, name, slug, type, order_by, has_archives }
```

#### Filter Helpers

```php
// Use with array_filter on product attributes
$variation_attributes = array_filter( $attributes, 'wc_attributes_array_filter_variation' );
$visible_attributes   = array_filter( $attributes, 'wc_attributes_array_filter_visible' );
```

#### Format Attribute Name for Variation Meta

```php
$meta_key = wc_variation_attribute_name( 'pa_color' );
// Returns: 'attribute_pa_color'
```

---

### 2.2 Product Functions

**File**: `woocommerce/includes/wc-product-functions.php`

#### Get Product Types

```php
$types = wc_get_product_types();
// Returns: ['simple' => 'Simple product', 'variable' => 'Variable product', ...]
```

#### Get Variation Attributes for a Specific Variation

```php
$attrs = wc_get_product_variation_attributes( $variation_id );
// Returns: ['attribute_pa_color' => 'red', 'attribute_pa_size' => 'large']
```

**Use in StoreSuite**: Load existing variation attributes when editing.

#### Format Variation Display String

```php
$formatted = wc_get_formatted_variation( $variation, true );
// Returns: "Color: Red, Size: Large" (flat text)

$formatted = wc_get_formatted_variation( $variation, false );
// Returns: "<dt>Color:</dt><dd>Red</dd><dt>Size:</dt><dd>Large</dd>" (HTML)
```

**Use in StoreSuite**: Display variation summary in the variation header row.

#### Stock Status Options

```php
$options = wc_get_product_stock_status_options();
// Returns: ['instock' => 'In stock', 'outofstock' => 'Out of stock', 'onbackorder' => 'On backorder']
```

**StoreSuite already uses this** in the product form (line 372 of `product-form.php`).

---

### 2.3 CRUD Classes

**You don't need to write raw SQL or direct `update_post_meta()` calls.** WooCommerce's CRUD classes handle everything:

#### Creating a Variable Product

```php
$product = new WC_Product_Variable( $existing_id_or_0 );
$product->set_name( 'My Variable Product' );
$product->set_status( 'publish' );

// Build attributes
$color_attr = new WC_Product_Attribute();
$color_attr->set_id( wc_attribute_taxonomy_id_by_name( 'pa_color' ) );
$color_attr->set_name( 'pa_color' );
$color_attr->set_options( [ $red_term_id, $blue_term_id ] ); // term IDs for taxonomy
$color_attr->set_visible( true );
$color_attr->set_variation( true );
$color_attr->set_position( 0 );

$custom_attr = new WC_Product_Attribute();
$custom_attr->set_id( 0 );                    // 0 = custom attribute
$custom_attr->set_name( 'Material' );
$custom_attr->set_options( [ 'Cotton', 'Polyester' ] ); // strings for custom
$custom_attr->set_visible( true );
$custom_attr->set_variation( true );
$custom_attr->set_position( 1 );

$product->set_attributes( [ $color_attr, $custom_attr ] );
$product_id = $product->save();
```

**StoreSuite's `ProductManager::create_product()` already does**: `$product->set_attributes( $args['attributes'] )` at line 322. You just need to prepare the `WC_Product_Attribute` array before passing it.

#### Creating a Variation

```php
$variation = new WC_Product_Variation();
$variation->set_parent_id( $product_id );
$variation->set_attributes( [
    'pa_color' => 'red',      // term slug for taxonomy
    'material' => 'Cotton',   // text for custom (use sanitize_title key)
] );
$variation->set_regular_price( '29.99' );
$variation->set_sale_price( '24.99' );
$variation->set_manage_stock( true );
$variation->set_stock_quantity( 50 );
$variation->set_stock_status( 'instock' );
$variation->set_weight( '0.5' );
$variation->set_length( '10' );
$variation->set_width( '5' );
$variation->set_height( '2' );
$variation->set_image_id( $attachment_id );
$variation->set_description( 'Red cotton variant' );
$variation->set_sku( 'PROD-RED-COT' );
$variation->set_status( 'publish' );       // 'publish' = enabled, 'private' = disabled
$variation->save();
```

#### Syncing Parent After Variation Changes

```php
// CRITICAL: Always call after creating/updating/deleting variations
WC_Product_Variable::sync( $product_id );
// This syncs: min/max prices, stock status, clears transients
```

**StoreSuite's `ProductManager` already calls this** at line 639.

#### Reading Variation Data

```php
$variable_product = wc_get_product( $product_id );

// Get all variation IDs
$children = $variable_product->get_children();             // All IDs
$visible  = $variable_product->get_visible_children();     // Visible only

// Get variation attributes with their possible values
$variation_attrs = $variable_product->get_variation_attributes();
// Returns: ['pa_color' => ['red', 'blue'], 'pa_size' => ['s', 'm', 'l']]

// Get price range
$min_price = $variable_product->get_variation_price( 'min' );
$max_price = $variable_product->get_variation_price( 'max' );

// Get all variation data (formatted for frontend)
$available = $variable_product->get_available_variations();
// Returns array of arrays with: variation_id, attributes, price_html, image, etc.

// Get as objects (cheaper, no HTML generation)
$variation_objects = $variable_product->get_available_variations( 'objects' );
```

#### Reading a Single Variation

```php
$variation = wc_get_product( $variation_id );
$variation->get_regular_price();
$variation->get_sale_price();
$variation->get_stock_status();
$variation->get_manage_stock();
$variation->get_stock_quantity();
$variation->get_weight();
$variation->get_sku();
$variation->get_image_id();
$variation->get_description();
$variation->get_attributes();                        // ['pa_color' => 'red']
$variation->get_variation_attributes();               // ['attribute_pa_color' => 'red']
$variation->get_parent_id();
$variation->get_status();                            // 'publish' or 'private'
```

#### Deleting a Variation

```php
$variation = wc_get_product( $variation_id );
$parent_id = $variation->get_parent_id();
$variation->delete( true );  // true = force delete (bypass trash)
WC_Product_Variable::sync( $parent_id );
```

---

### 2.4 Data Store Methods

#### Find Matching Variation by Attributes

```php
$data_store   = WC_Data_Store::load( 'product' );
$variation_id = $data_store->find_matching_product_variation(
    wc_get_product( $product_id ),
    [
        'attribute_pa_color' => 'red',
        'attribute_pa_size'  => 'large',
    ]
);
// Returns: variation ID (int) or 0 if not found
```

**Use in StoreSuite**: Check for duplicate variations before generating.

---

### 2.5 Template Functions

**File**: `woocommerce/includes/wc-template-functions.php`

#### Render Attribute Dropdown

```php
wc_dropdown_variation_attribute_options( [
    'options'          => $options,        // Array of slugs/values
    'attribute'        => 'pa_color',      // Attribute taxonomy name
    'product'          => $product,        // WC_Product_Variable object
    'selected'         => $selected_value, // Pre-selected value
    'name'             => 'attribute_pa_color', // HTML name attribute
    'id'               => 'pa_color',      // HTML id attribute
    'class'            => 'msf-form-control', // Custom CSS class
    'show_option_none' => __( 'Choose an option', 'storesuite' ),
] );
```

**Use in StoreSuite**: Not needed for the product add/edit form — this is a customer-facing storefront function. For the edit form, use the attribute dropdown pattern from WC's `html-variation-admin.php` (see Section 9.6).

---

## 3. WooCommerce AJAX Endpoints You Can Reuse

### 3.1 Directly Reusable

#### `json_search_products_and_variations` — Search Products

| Property | Value |
|---|---|
| **Action** | `woocommerce_json_search_products_and_variations` |
| **Method** | GET via AJAX |
| **Auth Required** | Yes (`edit_products` capability) |
| **Nonce** | `search-products` |
| **Parameters** | `term` (search string), `limit` (optional) |
| **Returns** | JSON `{ id: "name (#id)" }` |

**StoreSuite already uses this** for upsell/cross-sell product search (line 445 of `product-form.php`).

#### `json_search_products` — Search Products Only

Same as above but excludes variations. StoreSuite also uses this via select2 `data-action`.

---

### 3.2 Admin-Only (Must Build Your Own)

These WooCommerce AJAX endpoints **cannot be reused** on the frontend because they:
1. Require `edit_products` capability (admin only)
2. Return HTML designed for WooCommerce's admin meta boxes
3. Depend on admin-specific nonces

| WC AJAX Action | Why Can't Reuse | StoreSuite Alternative |
|---|---|---|
| `woocommerce_load_variations` | Returns admin meta box HTML | Build `storesuite_load_variations` |
| `woocommerce_save_variations` | Calls `WC_Meta_Box_Product_Data::save_variations()` — admin-specific | Build `storesuite_save_variations` using WC CRUD |
| `woocommerce_add_variation` | Returns admin variation row HTML | Build `storesuite_add_variation` returning your template |
| `woocommerce_remove_variations` | Admin nonce + capability check | Build `storesuite_remove_variation` |
| `woocommerce_bulk_edit_variations` | Admin-only | Build if needed, or skip |
| `woocommerce_add_attributes_and_variations` | Admin-only | Build `storesuite_generate_variations` |

**However**, inside your custom AJAX handlers, you can call all the WC PHP functions and CRUD classes listed in Section 2. You're only building the thin AJAX wrapper — the heavy lifting is done by WooCommerce:

```php
// Your custom AJAX handler — but using WC classes internally
public function handle_save_variations() {
    check_ajax_referer( 'storesuite_variation_nonce', '_wpnonce' );

    // Use WC_Product_Variation CRUD — no raw SQL needed
    $variation = new WC_Product_Variation( $variation_id );
    $variation->set_regular_price( wc_format_decimal( $_POST['price'] ) );
    $variation->set_attributes( $attrs );
    $variation->save();

    // Use WC sync — no manual price aggregation
    WC_Product_Variable::sync( $product_id );

    wp_send_json_success();
}
```

---

## 4. JavaScript Assets You Can Enqueue

### 4.1 Already Used by StoreSuite

| Script Handle | WC File | StoreSuite Handle |
|---|---|---|
| `selectWoo` | `assets/js/selectWoo/selectWoo.full.js` | `storesuite_selectWoo` (re-registered) |
| `wc-accounting` | `assets/js/accounting/accounting.min.js` | `wc-accounting` (direct) |

These are already registered and enqueued in `Assets.php`.

### 4.2 Ready to Enqueue (Frontend-Safe)

These WooCommerce scripts work on the frontend and can be enqueued directly:

#### `jquery-blockui` — Loading Overlay

```php
wp_enqueue_script( 'jquery-blockui' );
// or the WC handle:
wp_enqueue_script( 'wc-jquery-blockui' );
```

```javascript
// Usage:
$('#element').block({ message: null, overlayCSS: { background: '#fff', opacity: 0.6 } });
$('#element').unblock();
```

**Note**: StoreSuite already has `window.StoreSuite.msfcLoader.block()` / `.unblock()`. You may not need this.

#### `wc-enhanced-select` — Searchable Select Dropdowns

```php
wp_enqueue_script( 'wc-enhanced-select' );
wp_enqueue_style( 'select2' );
```

This provides AJAX-powered product/customer/category search selects. Initializes on elements with class `.wc-product-search`, `.wc-customer-search`, etc.

**StoreSuite already uses this pattern** for upsell/cross-sell selects via `data-action="woocommerce_json_search_products_and_variations"`.

**Can reuse for**: Searching attribute terms if you have many terms in a taxonomy.

#### `wc-backbone-modal` — Modal Dialogs

```php
wp_enqueue_script( 'wc-backbone-modal' );
```

```javascript
$( this ).WCBackboneModal({ template: 'wc-modal-add-attribute' });
```

**Use in StoreSuite**: Bulk price edit modals, confirmation dialogs. However, StoreSuite already uses SweetAlert2, which is likely sufficient.

#### `jquery-tiptip` — Tooltips

```php
wp_enqueue_script( 'wc-jquery-tiptip' );
```

```javascript
$( '.tips' ).tipTip({ attribute: 'data-tip', fadeIn: 50, fadeOut: 50 });
```

**Use in StoreSuite**: Tooltip help icons next to variation fields (e.g., "Leave blank to use parent SKU").

#### `serializejson` — Form Serialization to JSON

```php
wp_enqueue_script( 'wc-serializejson' );
```

```javascript
var data = $( ':input', '.variations-container' ).serializeJSON();
// Returns nested object from form fields
```

**Use in StoreSuite**: Serialize variation form data before AJAX save. Cleaner than `FormData` for complex nested arrays.

### 4.3 Admin-Only (Extract Logic Instead)

These scripts are **NOT reusable** on a frontend form because they:
- Depend on admin-specific DOM selectors (`#variable_product_options`, `#woocommerce-product-data`, `.woocommerce_variation`)
- Call admin-only AJAX endpoints
- Require admin localized data (`woocommerce_admin_meta_boxes_variations`)

| Script Handle | Why Not Reusable |
|---|---|
| `wc-admin-variation-meta-boxes` | Selectors: `#variable_product_options`, `.woocommerce_variations`, `#woocommerce-product-data` |
| `wc-admin-product-meta-boxes` | Selectors: `#product_attributes`, `.product_attributes` |
| `wc-admin-meta-boxes` | Admin meta box framework |

**However**, you can study and extract the **logic patterns** from these scripts:

| Pattern from WC Admin JS | Reuse in StoreSuite JS |
|---|---|
| Variation pagination logic | Copy the page-nav approach into your custom JS |
| Change tracking (`.variation-needs-update` class) | Implement similar change-detection in your form |
| AJAX save only changed variations | Replicate the "dirty flag" pattern |
| Generate all combinations algorithm | Port `link_all_variations()` logic to your AJAX handler (PHP) |
| Image upload via `wp.media` | Same API — just change DOM selectors |

---

## 5. CSS You Can Reuse

StoreSuite has its own design system (`msf-card`, `msf-form-control`, `my-storesuite-button`), so wholesale reuse of WC admin CSS doesn't make sense. But selective reuse can help:

#### Select2/SelectWoo Styles

```php
wp_enqueue_style( 'select2' );
// WC file: assets/css/select2.css
```

StoreSuite already loads select2 styles. No change needed.

#### jQuery UI Datepicker

```php
wp_enqueue_style( 'storesuite_jquery-ui-style' );
// Already registered in Assets.php pointing to WC's jquery-ui.min.css
```

Already used by StoreSuite for sale price date pickers.

---

## 6. Practical Implementation

### The Recommended Approach: WC PHP + Custom AJAX + Custom JS

```
┌─────────────────────────────────────────────────────────────────────┐
│                     WHAT TO REUSE FROM WOOCOMMERCE                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  PHP (FULL REUSE):                                                  │
│  ├── wc_get_attribute_taxonomies()     → populate attribute dropdown│
│  ├── wc_attribute_label()              → display attribute names    │
│  ├── wc_attribute_taxonomy_name()      → build taxonomy names       │
│  ├── wc_attribute_taxonomy_id_by_name()→ get IDs for WC_Product_Attr│
│  ├── wc_get_text_attributes()          → parse custom attr values   │
│  ├── wc_get_product_stock_status_options() → stock status dropdown  │
│  ├── wc_get_product_variation_attributes() → load variation attrs   │
│  ├── wc_get_formatted_variation()      → display variation summary  │
│  ├── WC_Product_Attribute class        → build attribute objects    │
│  ├── WC_Product_Variable class         → get_children(), sync()     │
│  ├── WC_Product_Variation class        → full CRUD (set/get/save)   │
│  ├── WC_Product_Variable::sync()       → sync prices/stock to parent│
│  └── WC_Data_Store::find_matching_product_variation()              │
│                                                                     │
│  JS (REUSE DIRECTLY):                                               │
│  ├── selectWoo          → already used by StoreSuite               │
│  ├── wc-accounting      → already used by StoreSuite               │
│  ├── jquery-blockui     → optional (StoreSuite has own loader)     │
│  ├── wc-serializejson   → serialize variation form data            │
│  └── wp.media           → WordPress core, variation image upload   │
│                                                                     │
├─────────────────────────────────────────────────────────────────────┤
│                     WHAT TO BUILD CUSTOM                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  PHP (AJAX WRAPPERS ONLY):                                          │
│  ├── storesuite_save_attributes       → calls WC_Product_Attribute │
│  ├── storesuite_load_variations       → calls get_children() + tpl │
│  ├── storesuite_add_variation         → calls new WC_Product_Var   │
│  ├── storesuite_generate_variations   → combination loop + CRUD    │
│  ├── storesuite_save_variations       → loop + WC_Product_Var CRUD │
│  └── storesuite_remove_variation      → $variation->delete()       │
│                                                                     │
│  JS (CUSTOM UI):                                                    │
│  ├── Attribute add/remove/save UI     → custom DOM, WC AJAX wrapper│
│  ├── Variation list/pagination        → custom DOM, AJAX loader    │
│  ├── Variation expand/collapse        → simple jQuery toggle       │
│  ├── Product type toggle (show/hide)  → show_if_variable logic     │
│  └── Variation image upload           → wp.media (WP core reuse)   │
│                                                                     │
│  Templates:                                                         │
│  ├── product-attribute.php            → StoreSuite card styling    │
│  ├── product-variations.php           → wrapper with buttons       │
│  └── product-variation.php            → single variation row       │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 7. Code Examples

### Example 1: Populate Attribute Dropdown Using WC Functions

```php
// In your template — zero custom code for data fetching
$attribute_taxonomies = wc_get_attribute_taxonomies();
?>
<select id="storesuite-add-attribute-select" class="msf-form-control">
    <option value=""><?php esc_html_e( 'Custom Attribute', 'storesuite' ); ?></option>
    <?php foreach ( $attribute_taxonomies as $tax ) : ?>
        <option value="<?php echo esc_attr( wc_attribute_taxonomy_name( $tax->attribute_name ) ); ?>">
            <?php echo esc_html( $tax->attribute_label ); ?>
        </option>
    <?php endforeach; ?>
</select>
```

### Example 2: Build Attributes Using WC Classes (Save Handler)

```php
// In your AJAX handler — WC does all the heavy lifting
$attribute = new WC_Product_Attribute();
$attribute->set_id( wc_attribute_taxonomy_id_by_name( $taxonomy_name ) ); // WC function
$attribute->set_name( $taxonomy_name );
$attribute->set_visible( true );
$attribute->set_variation( true );

// Get term IDs from slugs — WC handles the lookup
$term_ids = [];
foreach ( $slugs as $slug ) {
    $term = get_term_by( 'slug', $slug, $taxonomy_name );
    if ( $term ) {
        $term_ids[] = $term->term_id;
    }
}
$attribute->set_options( $term_ids );

$product->set_attributes( [ $attribute ] );
$product->save(); // WC handles serialization, meta storage, term relationships
```

### Example 3: Generate Variations Using WC CRUD

```php
// In your AJAX handler — one WC call per variation
foreach ( $combinations as $combo ) {
    $variation = new WC_Product_Variation();
    $variation->set_parent_id( $product_id );
    $variation->set_attributes( $combo ); // WC handles attribute_ prefix internally
    $variation->set_status( 'publish' );
    $variation->save(); // WC handles post creation, meta storage
}

// One call syncs everything
WC_Product_Variable::sync( $product_id ); // WC syncs prices, stock, transients
```

### Example 4: Load Variations Using WC Methods

```php
// In your AJAX handler — WC provides all data
$product  = wc_get_product( $product_id );
$children = $product->get_children(); // WC queries and caches this

foreach ( array_slice( $children, $offset, $per_page ) as $variation_id ) {
    $variation = wc_get_product( $variation_id );
    
    // All getters available:
    $price       = $variation->get_regular_price();   // WC getter
    $sku         = $variation->get_sku();              // WC getter (inherits parent if empty)
    $stock       = $variation->get_stock_status();     // WC getter
    $image       = $variation->get_image_id();         // WC getter
    $attrs       = $variation->get_attributes();       // WC getter: ['pa_color' => 'red']
    $description = $variation->get_description();      // WC getter
    $enabled     = $variation->get_status() === 'publish'; // WC getter
    
    // Pass to your template
}
```

### Example 5: Check for Duplicate Before Generating

```php
$data_store = WC_Data_Store::load( 'product' );
$existing   = $data_store->find_matching_product_variation(
    $product,
    [
        'attribute_pa_color' => 'red',
        'attribute_pa_size'  => 'large',
    ]
);

if ( $existing ) {
    // Skip — variation already exists
}
```

### Example 6: Enqueue Additional WC Scripts

```php
// In Assets.php enqueue_front_scripts() — add these for variation support
wp_enqueue_script( 'wc-serializejson' );

// If you want WC's enhanced select for attribute term search:
wp_enqueue_script( 'wc-enhanced-select' );
wp_localize_script( 'wc-enhanced-select', 'wc_enhanced_select_params', [
    'ajax_url'                       => admin_url( 'admin-ajax.php' ),
    'search_products_nonce'          => wp_create_nonce( 'search-products' ),
    'search_taxonomy_terms_nonce'    => wp_create_nonce( 'search-taxonomy-terms' ),
    'i18n_searching'                 => __( 'Searching...', 'storesuite' ),
    'i18n_no_matches'                => __( 'No matches found', 'storesuite' ),
    'i18n_input_too_short_1'         => __( 'Please enter 1 or more characters', 'storesuite' ),
] );
```

---

## 8. What StoreSuite Already Reuses

| What | WC Source | StoreSuite Location |
|---|---|---|
| `selectWoo` JS | `WC()->plugin_url() . '/assets/js/selectWoo/selectWoo.full.js'` | `Assets.php` line 103 |
| `wc-accounting` JS | WC registered handle | `Assets.php` line 303 |
| jQuery UI CSS | `WC()->plugin_url() . '/assets/css/jquery-ui/jquery-ui.min.css'` | `Assets.php` line 125 |
| `wc_get_product()` | WC core function | `ProductManager.php`, `ProductController.php` |
| `WC_Product_Variable::sync()` | WC core class | `ProductManager.php` line 639 |
| `wc_attribute_taxonomy_name_by_id()` | WC attribute function | `ProductManager.php` line 694 |
| `wc_get_product_types()` | WC product function | `product-form.php` line 443 |
| `wc_get_product_stock_status_options()` | WC product function | `product-form.php` line 372 |
| `WC_Product_Factory::get_classname_from_product_type()` | WC factory | `ProductManager.php` line 229 |
| `wc_format_decimal()` | WC formatting | `ProductManager.php` throughout |
| `wc_stock_amount()` | WC formatting | `ProductManager.php` throughout |
| `wc_clean()` | WC sanitization | `ProductManager.php` throughout |

**Bottom line**: StoreSuite's `ProductManager` is already well-integrated with WooCommerce's API. Adding variable product support is mostly about building the UI templates, AJAX wrappers, and JavaScript — while letting WC's PHP classes do all the actual data handling.

---

## Quick Reference: Functions You'll Use Most

| Task | WC Function/Class | Custom Code Needed? |
|---|---|---|
| List attribute taxonomies | `wc_get_attribute_taxonomies()` | No |
| Get attribute label | `wc_attribute_label()` | No |
| Get taxonomy ID | `wc_attribute_taxonomy_id_by_name()` | No |
| Build attribute object | `new WC_Product_Attribute()` + setters | No |
| Save attributes to product | `$product->set_attributes()` + `save()` | No |
| Create variation | `new WC_Product_Variation()` + setters + `save()` | No |
| Update variation | `wc_get_product( $id )` + setters + `save()` | No |
| Delete variation | `$variation->delete( true )` | No |
| Get all variation IDs | `$product->get_children()` | No |
| Sync parent prices/stock | `WC_Product_Variable::sync()` | No |
| Find duplicate variation | `$data_store->find_matching_product_variation()` | No |
| Stock status options | `wc_get_product_stock_status_options()` | No |
| Parse custom attr values | `wc_get_text_attributes()` | No |
| **AJAX endpoints** | — | **Yes** (thin wrappers) |
| **UI templates** | — | **Partial** (see Section 9) |
| **JavaScript interactions** | — | **Yes** (custom DOM) |

---

## 9. Reusing WooCommerce HTML Templates

WooCommerce ships several PHP template/view files and form-helper functions that generate ready-to-use HTML. Rather than hand-coding every `<input>`, `<select>`, and `<label>`, StoreSuite can call these functions directly. This section covers what's reusable, what's not, and exactly how to wire it up.

### 9.1 Reuse Verdict at a Glance

| WC Template / Function | Reusable on Frontend? | Notes |
|---|---|---|
| `woocommerce_wp_text_input()` | **Yes** | Generates `<p class="form-field"><label><input></p>`. Works anywhere. |
| `woocommerce_wp_select()` | **Yes** | Generates `<p class="form-field"><label><select></p>`. Works anywhere. |
| `woocommerce_wp_textarea_input()` | **Yes** | Generates `<p class="form-field"><label><textarea></p>`. Works anywhere. |
| `woocommerce_wp_checkbox()` | **Yes** | Generates `<p class="form-field"><label><input type="checkbox"></p>`. |
| `woocommerce_wp_hidden_input()` | **Yes** | Generates hidden input. |
| `wc_help_tip()` | **Yes** | Generates tooltip `<span>` with `?` icon. |
| `wp_dropdown_categories()` | **Yes** | WordPress core. Used for shipping class dropdown. |
| `html-variation-admin.php` | **Partially** | Can be included directly but depends on `$product_object`, `$variation_object`, `$loop`, `$variation_data`, `$variation` variables being set. Admin-styled. |
| `html-product-attribute-inner.php` | **Partially** | Renders attribute name/value/checkbox table. Admin-styled. |
| `html-product-data-variations.php` | **No** | Tightly coupled to admin meta box structure, admin images, Backbone JS. |
| `html-product-data-attributes.php` | **No** | Depends on admin meta box wrapper, admin search widget. |

### 9.2 Form Helper Functions (Full Reuse)

**File**: `woocommerce/includes/admin/wc-meta-box-functions.php`

These functions are loaded when WooCommerce is active. They output standalone HTML form fields and work perfectly on the frontend. The only requirement: make sure the file is loaded.

#### Ensuring the functions are available

On admin pages they're auto-loaded. On the frontend, include them explicitly:

```php
// In your AJAX handler or template loader, ensure the file is loaded
if ( ! function_exists( 'woocommerce_wp_text_input' ) ) {
    include_once WC_ABSPATH . 'includes/admin/wc-meta-box-functions.php';
}
```

#### `woocommerce_wp_text_input( $field )` — Text, Number, Price Fields

```php
woocommerce_wp_text_input( array(
    'id'            => "variable_regular_price_{$loop}",
    'name'          => "variable_regular_price[{$loop}]",
    'value'         => $variation_object->get_regular_price( 'edit' ),
    'label'         => sprintf( __( 'Regular price (%s)', 'storesuite' ), get_woocommerce_currency_symbol() ),
    'data_type'     => 'price',          // Auto-adds wc_input_price class + formats value
    'wrapper_class' => 'form-row form-row-first',
    'placeholder'   => __( 'Variation price (required)', 'storesuite' ),
) );
```

**Output HTML:**

```html
<p class="form-field variable_regular_price_0_field form-row form-row-first">
    <label for="variable_regular_price_0">Regular price ($)</label>
    <input type="text" class="short wc_input_price" name="variable_regular_price[0]"
           id="variable_regular_price_0" value="29.99" placeholder="Variation price (required)">
</p>
```

**Supported `data_type` values:**

| `data_type` | CSS class added | Value formatting |
|---|---|---|
| `'price'` | `wc_input_price` | `wc_format_localized_price()` |
| `'decimal'` | `wc_input_decimal` | `wc_format_localized_decimal()` |
| `'stock'` | `wc_input_stock` | `wc_stock_amount()` |
| `'url'` | `wc_input_url` | `esc_url()` |

#### `woocommerce_wp_select( $field )` — Select Dropdowns

```php
woocommerce_wp_select( array(
    'id'            => "variable_stock_status{$loop}",
    'name'          => "variable_stock_status[{$loop}]",
    'value'         => $variation_object->get_stock_status( 'edit' ),
    'label'         => __( 'Stock status', 'storesuite' ),
    'options'       => wc_get_product_stock_status_options(),
    'desc_tip'      => true,
    'description'   => __( 'Controls whether the product is listed as "in stock" or "out of stock".', 'storesuite' ),
    'wrapper_class' => 'form-row form-row-full',
) );
```

**Output HTML:**

```html
<p class="form-field variable_stock_status0_field form-row form-row-full">
    <label for="variable_stock_status0">Stock status</label>
    <span class="woocommerce-help-tip" data-tip="Controls whether..."></span>
    <select name="variable_stock_status[0]" id="variable_stock_status0" class="select short">
        <option value="instock" selected>In stock</option>
        <option value="outofstock">Out of stock</option>
        <option value="onbackorder">On backorder</option>
    </select>
</p>
```

#### `woocommerce_wp_textarea_input( $field )` — Textareas

```php
woocommerce_wp_textarea_input( array(
    'id'            => "variable_description{$loop}",
    'name'          => "variable_description[{$loop}]",
    'value'         => $variation_object->get_description( 'edit' ),
    'label'         => __( 'Description', 'storesuite' ),
    'desc_tip'      => true,
    'description'   => __( 'Enter an optional description for this variation.', 'storesuite' ),
    'wrapper_class' => 'form-row form-row-full',
) );
```

#### `woocommerce_wp_checkbox( $field )` — Checkboxes

```php
woocommerce_wp_checkbox( array(
    'id'            => "variable_enabled{$loop}",
    'name'          => "variable_enabled[{$loop}]",
    'label'         => __( 'Enabled', 'storesuite' ),
    'value'         => $variation_object->get_status( 'edit' ) === 'publish' ? 'yes' : 'no',
    'cbvalue'       => 'yes',
    'wrapper_class' => 'form-row',
) );
```

#### `wc_help_tip( $tip_text )` — Tooltip Icons

```php
echo wc_help_tip( __( 'Leave blank to use the parent product SKU.', 'storesuite' ) );
```

**Output:** `<span class="woocommerce-help-tip" tabindex="0" data-tip="Leave blank..."></span>`

### 9.3 Including `html-variation-admin.php` Directly

WooCommerce's single-variation template (`html-variation-admin.php`) can be included on the frontend if you set up the required variables. This gives you the **exact same variation form** used in WooCommerce admin.

#### Required Variables

The template expects these to exist in scope:

```php
$variation_id     = $variation->ID;                      // Variation post ID
$variation        = get_post( $variation_id );           // WP_Post object
$variation_object = wc_get_product( $variation_id );     // WC_Product_Variation
$variation_data   = array();                             // Deprecated but still referenced
$product_object   = wc_get_product( $parent_id );       // WC_Product_Variable (parent)
$loop             = 0;                                   // Loop index (0, 1, 2...)
```

#### How to Include

```php
// Ensure WC admin functions are loaded
if ( ! function_exists( 'woocommerce_wp_text_input' ) ) {
    include_once WC_ABSPATH . 'includes/admin/wc-meta-box-functions.php';
}

$product_object = wc_get_product( $product_id );
$children       = $product_object->get_children();
$loop           = 0;

foreach ( $children as $variation_id ) {
    $variation_object = wc_get_product( $variation_id );
    $variation        = get_post( $variation_id );
    $variation_data   = array();

    // Include WooCommerce's own template
    include WC_ABSPATH . 'includes/admin/meta-boxes/views/html-variation-admin.php';

    $loop++;
}
```

#### What You Get

The full variation editing form with:
- Image upload area
- SKU + GTIN fields
- Enabled / Downloadable / Virtual / Manage Stock checkboxes
- Regular price + Sale price (with schedule)
- Stock quantity + Backorders + Low stock threshold
- Weight + Dimensions
- Shipping class dropdown
- Tax class dropdown
- Description textarea
- Downloadable files section
- All WooCommerce action hooks fire (`woocommerce_variation_options`, `woocommerce_variation_options_pricing`, etc.)

#### Caveats

| Issue | Solution |
|---|---|
| **Admin CSS classes** (`form-row`, `form-field`, `wc-metabox`) don't match StoreSuite styling | Add a CSS wrapper class and write overrides, OR use the approach in 9.5 |
| **Image upload JS** expects admin `upload_image_button` handler | WC's admin JS handles this — you'd need to load `wc-admin-variation-meta-boxes` or handle it yourself |
| **`$post` global** may not be set on frontend | Set it: `global $post; $post = get_post( $product_id );` |
| **Action hooks** fire — third-party plugins may inject admin-only content | Test carefully; filter if needed |

### 9.4 Including `html-product-attribute-inner.php` Directly

The attribute row inner template can also be included:

```php
// Required variables
$attribute = $product_object->get_attributes()['pa_color']; // WC_Product_Attribute
$i         = 0; // Attribute index

include WC_ABSPATH . 'includes/admin/meta-boxes/views/html-product-attribute-inner.php';
```

This generates:
- Name field (readonly for taxonomy, editable for custom)
- Values field (multi-select for taxonomy with Select all/Select none/Create value buttons, textarea for custom)
- "Visible on the product page" checkbox
- "Used for variations" checkbox (shown only for variable products via `.show_if_variable` class)

**But** the outer wrapper (`html-product-attribute.php`) uses admin meta box markup (`wc-metabox`, `postbox`, `handlediv`). You'd wrap the inner template with your own StoreSuite card instead.

### 9.5 Hybrid Approach: WC Functions + StoreSuite Styling

The most practical approach is to use WC's form helper functions inside your own StoreSuite templates. This gives you **WC's field logic** (data types, formatting, tooltips) with **StoreSuite's layout**:

```php
<?php
// templates/products/product-variation.php (StoreSuite template)

if ( ! function_exists( 'woocommerce_wp_text_input' ) ) {
    include_once WC_ABSPATH . 'includes/admin/wc-meta-box-functions.php';
}
?>
<div class="storesuite-variation-row msf-card msf-mb-12" data-variation-id="<?php echo esc_attr( $variation_id ); ?>">
    <!-- StoreSuite-styled header -->
    <div class="storesuite-variation-header">
        <strong>#<?php echo esc_html( $variation_id ); ?></strong>
        <?php
        // Reuse WC's exact attribute dropdown rendering pattern
        foreach ( $product_object->get_attributes( 'edit' ) as $attribute ) {
            if ( ! $attribute->get_variation() ) continue;

            $selected_value = isset( $attribute_values[ sanitize_title( $attribute->get_name() ) ] )
                ? $attribute_values[ sanitize_title( $attribute->get_name() ) ] : '';
            ?>
            <select class="msf-form-control" name="attribute_<?php echo esc_attr( sanitize_title( $attribute->get_name() ) . "[{$loop}]" ); ?>">
                <option value=""><?php printf( esc_html__( 'Any %s…', 'storesuite' ), esc_html( wc_attribute_label( $attribute->get_name() ) ) ); ?></option>
                <?php if ( $attribute->is_taxonomy() ) : ?>
                    <?php foreach ( $attribute->get_terms() as $option ) : ?>
                        <option <?php selected( $selected_value, $option->slug ); ?> value="<?php echo esc_attr( $option->slug ); ?>">
                            <?php echo esc_html( apply_filters( 'woocommerce_variation_option_name', $option->name, $option, $attribute->get_name(), $product_object ) ); ?>
                        </option>
                    <?php endforeach; ?>
                <?php else : ?>
                    <?php foreach ( $attribute->get_options() as $option ) : ?>
                        <option <?php selected( $selected_value, $option ); ?> value="<?php echo esc_attr( $option ); ?>">
                            <?php echo esc_html( apply_filters( 'woocommerce_variation_option_name', $option, null, $attribute->get_name(), $product_object ) ); ?>
                        </option>
                    <?php endforeach; ?>
                <?php endif; ?>
            </select>
            <?php
        }
        ?>
        <input type="hidden" name="variable_post_id[<?php echo esc_attr( $loop ); ?>]" value="<?php echo esc_attr( $variation_id ); ?>">
    </div>

    <!-- StoreSuite-styled body using WC form helpers -->
    <div class="storesuite-variation-body msf-card-content" style="display:none;">
        <div class="row">
            <div class="col-md-6">
                <?php
                // WC generates the full <p><label><input></p> block
                woocommerce_wp_text_input( array(
                    'id'            => "variable_regular_price_{$loop}",
                    'name'          => "variable_regular_price[{$loop}]",
                    'value'         => $variation_object->get_regular_price( 'edit' ),
                    'label'         => sprintf( __( 'Regular price (%s)', 'storesuite' ), get_woocommerce_currency_symbol() ),
                    'data_type'     => 'price',
                    'wrapper_class' => 'form-row form-row-first',
                    'placeholder'   => __( 'Variation price (required)', 'storesuite' ),
                ) );
                ?>
            </div>
            <div class="col-md-6">
                <?php
                woocommerce_wp_text_input( array(
                    'id'            => "variable_sale_price_{$loop}",
                    'name'          => "variable_sale_price[{$loop}]",
                    'value'         => $variation_object->get_sale_price( 'edit' ),
                    'label'         => sprintf( __( 'Sale price (%s)', 'storesuite' ), get_woocommerce_currency_symbol() ),
                    'data_type'     => 'price',
                    'wrapper_class' => 'form-row form-row-last',
                ) );
                ?>
            </div>
        </div>

        <div class="row">
            <div class="col-md-4">
                <?php
                woocommerce_wp_text_input( array(
                    'id'            => "variable_sku_{$loop}",
                    'name'          => "variable_sku[{$loop}]",
                    'value'         => $variation_object->get_sku( 'edit' ),
                    'label'         => __( 'SKU', 'storesuite' ),
                    'placeholder'   => $product_object->get_sku(),
                    'desc_tip'      => true,
                    'description'   => __( 'Leave blank to use the parent product SKU.', 'storesuite' ),
                    'wrapper_class' => 'form-row',
                ) );
                ?>
            </div>
            <div class="col-md-4">
                <?php
                woocommerce_wp_select( array(
                    'id'            => "variable_stock_status_{$loop}",
                    'name'          => "variable_stock_status[{$loop}]",
                    'value'         => $variation_object->get_stock_status( 'edit' ),
                    'label'         => __( 'Stock status', 'storesuite' ),
                    'options'       => wc_get_product_stock_status_options(),
                    'wrapper_class' => 'form-row',
                ) );
                ?>
            </div>
            <div class="col-md-4">
                <?php
                woocommerce_wp_text_input( array(
                    'id'            => "variable_stock_{$loop}",
                    'name'          => "variable_stock[{$loop}]",
                    'value'         => wc_stock_amount( $variation_object->get_stock_quantity( 'edit' ) ),
                    'label'         => __( 'Stock quantity', 'storesuite' ),
                    'type'          => 'number',
                    'data_type'     => 'stock',
                    'wrapper_class' => 'form-row show_if_variation_manage_stock',
                    'custom_attributes' => array( 'step' => 'any' ),
                ) );
                ?>
            </div>
        </div>

        <div class="row">
            <div class="col-md-3">
                <?php if ( wc_product_weight_enabled() ) :
                    woocommerce_wp_text_input( array(
                        'id'            => "variable_weight_{$loop}",
                        'name'          => "variable_weight[{$loop}]",
                        'value'         => wc_format_localized_decimal( $variation_object->get_weight( 'edit' ) ),
                        'label'         => sprintf( __( 'Weight (%s)', 'storesuite' ), get_option( 'woocommerce_weight_unit' ) ),
                        'data_type'     => 'decimal',
                        'placeholder'   => wc_format_localized_decimal( $product_object->get_weight() ),
                        'wrapper_class' => 'form-row',
                    ) );
                endif; ?>
            </div>
            <div class="col-md-9">
                <?php if ( wc_product_dimensions_enabled() ) : ?>
                    <p class="form-field dimensions_field">
                        <label><?php printf( esc_html__( 'Dimensions (L×W×H) (%s)', 'storesuite' ), esc_html( get_option( 'woocommerce_dimension_unit' ) ) ); ?></label>
                        <span class="wrap">
                            <input type="text" class="msf-form-control wc_input_decimal" name="variable_length[<?php echo esc_attr( $loop ); ?>]" value="<?php echo esc_attr( wc_format_localized_decimal( $variation_object->get_length( 'edit' ) ) ); ?>" placeholder="<?php echo esc_attr( wc_format_localized_decimal( $product_object->get_length() ) ); ?>">
                            <input type="text" class="msf-form-control wc_input_decimal" name="variable_width[<?php echo esc_attr( $loop ); ?>]" value="<?php echo esc_attr( wc_format_localized_decimal( $variation_object->get_width( 'edit' ) ) ); ?>" placeholder="<?php echo esc_attr( wc_format_localized_decimal( $product_object->get_width() ) ); ?>">
                            <input type="text" class="msf-form-control wc_input_decimal" name="variable_height[<?php echo esc_attr( $loop ); ?>]" value="<?php echo esc_attr( wc_format_localized_decimal( $variation_object->get_height( 'edit' ) ) ); ?>" placeholder="<?php echo esc_attr( wc_format_localized_decimal( $product_object->get_height() ) ); ?>">
                        </span>
                    </p>
                <?php endif; ?>
            </div>
        </div>

        <div class="row">
            <div class="col-md-12">
                <?php
                woocommerce_wp_textarea_input( array(
                    'id'            => "variable_description_{$loop}",
                    'name'          => "variable_description[{$loop}]",
                    'value'         => $variation_object->get_description( 'edit' ),
                    'label'         => __( 'Description', 'storesuite' ),
                    'wrapper_class' => 'form-row form-row-full',
                ) );
                ?>
            </div>
        </div>

        <?php
        // Fire WC hooks so third-party plugins can add fields
        do_action( 'woocommerce_variation_options_pricing', $loop, $variation_data, $variation );
        do_action( 'woocommerce_product_after_variable_attributes', $loop, $variation_data, $variation );
        ?>
    </div>
</div>
```

### 9.6 Reusing the Attribute Dropdown Rendering Pattern

WooCommerce's variation header template (`html-variation-admin.php` lines 23-55) uses `WC_Product_Attribute` objects to render attribute `<select>` dropdowns. This exact pattern is portable:

```php
// This pattern from WC can be copy-pasted with zero changes
foreach ( $product_object->get_attributes( 'edit' ) as $attribute ) {
    if ( ! $attribute->get_variation() ) {
        continue;
    }
    $selected_value = isset( $attribute_values[ sanitize_title( $attribute->get_name() ) ] )
        ? $attribute_values[ sanitize_title( $attribute->get_name() ) ] : '';
    ?>
    <select name="attribute_<?php echo esc_attr( sanitize_title( $attribute->get_name() ) . "[{$loop}]" ); ?>">
        <option value="">
            <?php printf( esc_html__( 'Any %s…', 'woocommerce' ), esc_html( wc_attribute_label( $attribute->get_name() ) ) ); ?>
        </option>
        <?php if ( $attribute->is_taxonomy() ) : ?>
            <?php foreach ( $attribute->get_terms() as $option ) : ?>
                <option <?php selected( $selected_value, $option->slug ); ?> value="<?php echo esc_attr( $option->slug ); ?>">
                    <?php echo esc_html( apply_filters( 'woocommerce_variation_option_name', $option->name, $option, $attribute->get_name(), $product_object ) ); ?>
                </option>
            <?php endforeach; ?>
        <?php else : ?>
            <?php foreach ( $attribute->get_options() as $option ) : ?>
                <option <?php selected( $selected_value, $option ); ?> value="<?php echo esc_attr( $option ); ?>">
                    <?php echo esc_html( apply_filters( 'woocommerce_variation_option_name', $option, null, $attribute->get_name(), $product_object ) ); ?>
                </option>
            <?php endforeach; ?>
        <?php endif; ?>
    </select>
    <?php
}
```

**This handles both taxonomy and custom attributes automatically**, applies WC filters, and uses the exact `name` format (`attribute_pa_color[0]`) that WC's save logic expects.

### 9.7 Reusing Shipping Class Dropdown

WooCommerce and WordPress core make it easy:

```php
wp_dropdown_categories( array(
    'taxonomy'         => 'product_shipping_class',
    'hide_empty'       => 0,
    'show_option_none' => __( 'Same as parent', 'storesuite' ),
    'name'             => "variable_shipping_class[{$loop}]",
    'id'               => '',
    'class'            => 'msf-form-control',
    'selected'         => $variation_object->get_shipping_class_id( 'edit' ),
) );
```

StoreSuite already uses this pattern in `product-form.php` line 416 for the parent product.

### 9.8 Reusing Tax Class Dropdown

```php
woocommerce_wp_select( array(
    'id'            => "variable_tax_class{$loop}",
    'name'          => "variable_tax_class[{$loop}]",
    'value'         => $variation_object->get_tax_class( 'edit' ),
    'label'         => __( 'Tax class', 'storesuite' ),
    'options'       => array( 'parent' => __( 'Same as parent', 'storesuite' ) ) + wc_get_product_tax_class_options(),
    'wrapper_class' => 'form-row',
) );
```

`wc_get_product_tax_class_options()` returns all WooCommerce tax classes — no need to query manually.

### 9.9 CSS Bridge: Making WC Form Fields Look Like StoreSuite

WC's form helpers output with classes like `form-field`, `form-row`, `form-row-first`, `form-row-last`, `short`. Add these overrides in StoreSuite's CSS:

```css
/* Bridge WC admin form classes to StoreSuite style */
.storesuite-variation-body .form-field,
.storesuite-variation-body .form-row {
    margin-bottom: 12px;
}

.storesuite-variation-body .form-field label,
.storesuite-variation-body .form-row label {
    display: block;
    font-weight: 500;
    margin-bottom: 4px;
}

.storesuite-variation-body .form-field input[type="text"],
.storesuite-variation-body .form-field input[type="number"],
.storesuite-variation-body .form-field select,
.storesuite-variation-body .form-field textarea,
.storesuite-variation-body .form-row input[type="text"],
.storesuite-variation-body .form-row input[type="number"],
.storesuite-variation-body .form-row select,
.storesuite-variation-body .form-row textarea {
    width: 100%;
    padding: 8px 12px;
    border: 1px solid #ddd;
    border-radius: 4px;
    /* Match StoreSuite's .msf-form-control */
}

.storesuite-variation-body .form-row-first {
    float: left;
    width: 48%;
}

.storesuite-variation-body .form-row-last {
    float: right;
    width: 48%;
}

.storesuite-variation-body .form-row-full {
    width: 100%;
    clear: both;
}

.storesuite-variation-body .woocommerce-help-tip {
    cursor: help;
    font-size: 12px;
}
```

### 9.10 Summary: Template Reuse Strategy

```
┌──────────────────────────────────────────────────────────────────┐
│  APPROACH A: Maximum WC Reuse (Quick but admin-looking)          │
│  ──────────────────────────────────────────────────────────────── │
│  Include html-variation-admin.php directly                       │
│  + Fastest to implement                                          │
│  + All WC hooks fire (plugin compatibility)                      │
│  - Looks like WP admin, not StoreSuite                           │
│  - Requires admin JS for image upload + toggle behavior          │
│  - Third-party hook output may break layout                      │
├──────────────────────────────────────────────────────────────────┤
│  APPROACH B: Hybrid (Recommended)                                │
│  ──────────────────────────────────────────────────────────────── │
│  StoreSuite template layout + WC form helper functions            │
│  + StoreSuite look and feel with Bootstrap grid                  │
│  + WC handles field formatting (prices, decimals, stock)         │
│  + WC filters applied (woocommerce_variation_option_name)        │
│  + Selective WC hooks for extension points                       │
│  + No admin JS dependency                                        │
│  - Need CSS bridge for WC form-field classes                     │
│  - ~50% less code than fully custom templates                    │
├──────────────────────────────────────────────────────────────────┤
│  APPROACH C: Fully Custom (Previous guide)                       │
│  ──────────────────────────────────────────────────────────────── │
│  100% custom HTML with raw <input>/<select> tags                 │
│  + Full control over markup and styling                          │
│  + No WC admin class dependencies                                │
│  - Most code to write                                            │
│  - Must manually handle price formatting, decimal separators     │
│  - Miss WC filters and hooks                                    │
└──────────────────────────────────────────────────────────────────┘

RECOMMENDATION: Approach B (Hybrid)
- Use StoreSuite templates for layout (your cards, grid, buttons)
- Call woocommerce_wp_text_input(), woocommerce_wp_select(), etc. for fields
- Copy WC's attribute dropdown rendering pattern (lines 23-55 from html-variation-admin.php)
- Fire selected WC hooks (woocommerce_variation_options_pricing, etc.) for plugin compat
- Add CSS bridge to style WC output within StoreSuite containers
```
