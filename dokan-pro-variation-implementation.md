# How Dokan Pro Implements Variable Product Variations on the Frontend

> **Goal**: Document exactly how Dokan Pro builds its vendor-facing variation management UI — the PHP handlers, JS architecture, templates, and data flow — so StoreSuite can replicate or adapt the same patterns.  
> **Generated**: February 24, 2026

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [File Map](#2-file-map)
3. [Two Design Generations](#3-two-design-generations)
4. [How Attributes Work](#4-how-attributes-work)
   - [Template Rendering](#41-template-rendering)
   - [Adding a New Attribute (AJAX)](#42-adding-a-new-attribute-ajax)
   - [Saving Attributes (AJAX)](#43-saving-attributes-ajax)
   - [JS Template for New Attribute Rows](#44-js-template-for-new-attribute-rows)
5. [How Variations Work](#5-how-variations-work)
   - [Variation Panel Output](#51-variation-panel-output)
   - [Loading Variations (AJAX Pagination)](#52-loading-variations-ajax-pagination)
   - [Adding a Single Variation (AJAX)](#53-adding-a-single-variation-ajax)
   - [Generate All Variations ("Link All")](#54-generate-all-variations-link-all)
   - [Saving Variations (AJAX)](#55-saving-variations-ajax)
   - [Removing a Variation (AJAX)](#56-removing-a-variation-ajax)
   - [Bulk Edit Variations (AJAX)](#57-bulk-edit-variations-ajax)
6. [JavaScript Architecture](#6-javascript-architecture)
   - [Three JS Objects](#61-three-js-objects)
   - [Dirty Tracking & Save-on-Submit](#62-dirty-tracking--save-on-submit)
   - [Pagination System](#63-pagination-system)
   - [Vendor Earning Calculation](#64-vendor-earning-calculation)
7. [Variation Template (Single Row)](#7-variation-template-single-row)
8. [Nonces & Localized Data](#8-nonces--localized-data)
9. [Key WooCommerce Reuse Points](#9-key-woocommerce-reuse-points)
10. [Data Flow Diagrams](#10-data-flow-diagrams)
11. [Comparison: Old vs New Dokan Attribute UI](#11-comparison-old-vs-new-dokan-attribute-ui)

---

## 1. Architecture Overview

Dokan Pro splits its variation feature across **three layers**:

```
┌────────────────────────────────────────────────────────────────────────┐
│  LAYER 1: PHP — Hook Registration & AJAX Handlers                     │
│                                                                        │
│  dokan-pro/includes/Products.php                                       │
│    → Hooks into `dokan_product_edit_after_inventory_variants`           │
│    → Loads the attribute + variation template section                   │
│                                                                        │
│  dokan-pro/includes/Ajax.php                                           │
│    → Registers wp_ajax_* hooks for all variation AJAX operations        │
│    → Contains the actual PHP logic for load/save/add/remove/bulk       │
│                                                                        │
│  dokan-pro/includes/functions-wc.php                                   │
│    → dokan_save_variations() — legacy save handler (manual loop)        │
│    → dokan_variable_product_type_options() — legacy UI (not used now)   │
├────────────────────────────────────────────────────────────────────────┤
│  LAYER 2: JavaScript — UI Interaction & AJAX Calls                     │
│                                                                        │
│  dokan-pro/assets/src/js/product-variation.js                          │
│    → Dokan_Product_Variation_Actions (UI toggle/sort/expand)            │
│    → Dokan_Product_Variation_Ajax (CRUD operations)                     │
│    → Dokan_Product_Variation_PageNav (pagination)                       │
│                                                                        │
│  dokan-pro/assets/src/js/product.js                                    │
│    → Attribute management (add/remove/save from old design)             │
├────────────────────────────────────────────────────────────────────────┤
│  LAYER 3: Templates — HTML Output                                      │
│                                                                        │
│  templates/products/product-variation.php       (main section wrapper)  │
│  templates/products/edit/html-product-attribute.php (single attr row)   │
│  templates/products/edit/html-product-variation.php (single variation)  │
│  templates/products/edit/html-product-variation-download.php            │
│  templates/products/edit/tmpl-add-attribute.php (JS <script> template)  │
│  templates/products/edit/attributes.php         (old design, legacy)    │
└────────────────────────────────────────────────────────────────────────┘
```

---

## 2. File Map

| File | Role |
|---|---|
| `includes/Products.php` | Hooks `load_variations_content()` and `load_variations_js_template()` into the product edit form |
| `includes/Ajax.php` | All `wp_ajax_dokan_*` handlers — the core backend logic |
| `includes/functions-wc.php` | `dokan_save_variations()` (legacy per-field save loop), `dokan_variable_product_type_options()` (legacy UI) |
| `assets/src/js/product-variation.js` | Modern variation JS — 3 objects handling actions, AJAX, pagination |
| `assets/src/js/product.js` | Attribute add/remove/save JS (old design references `#variants-holder`) |
| `templates/products/product-variation.php` | Main wrapper: attributes section + variation container |
| `templates/products/edit/html-product-attribute.php` | Single attribute row (collapsible, with name/values/checkboxes) |
| `templates/products/edit/html-product-variation.php` | Single variation row (collapsible, with all fields) |
| `templates/products/edit/tmpl-add-attribute.php` | `<script type="text/html" id="tmpl-dokan-add-attribute">` for cloning |
| `templates/products/edit/attributes.php` | Legacy attribute UI (simpler, Bootstrap-based) |
| `dokan-lite/includes/Product/functions.php` | `dokan_product_output_variations()` — the variation panel with toolbar + pagination |
| `dokan-lite/includes/Assets.php` | Localizes nonces and `variations_per_page` into `dokan` JS object |

---

## 3. Two Design Generations

Dokan has **two versions** of the attribute UI co-existing:

### Old Design (`templates/products/edit/attributes.php`)
- Uses `#variants-holder` with `.inputs-box` divs
- Bootstrap-style buttons (`btn btn-success`, `btn btn-danger`)
- Attribute values as individual `<input type="text">` fields with +/- buttons
- Saves via `dokan_save_attributes` AJAX (serializes `#variants-holder` form data)
- JS in `product.js` → `this.variants.save()`

### New Design (`templates/products/product-variation.php` + `html-product-attribute.php`)
- Uses `.dokan-attribute-option-list` with `<li>` items
- Dokan's own UI framework (`dokan-btn`, `dokan-form-control`, `dokan-section-heading`)
- Taxonomy attributes use **Select2/SelectWoo** multi-select (`dokan-select2`)
- Custom attributes use Select2 with `data-tags="true"` for free-text entry
- Collapsible attribute rows with toggle icons
- Adds attributes via `dokan_get_pre_attribute` AJAX (returns rendered HTML)
- "Select all" / "Select none" buttons for taxonomy terms
- "Add new" button to create new taxonomy terms inline via `dokan_add_new_attribute` AJAX

**The new design is what the current Dokan Pro product edit form uses.**

---

## 4. How Attributes Work

### 4.1 Template Rendering

**Entry point**: `Products::load_variations_content()` hooks at priority 20 on `dokan_product_edit_after_inventory_variants`.

```php
// includes/Products.php line 98-116
public function load_variations_content( $post, $post_id ) {
    $_has_attribute       = get_post_meta( $post_id, '_has_attribute', true );
    $_create_variations   = get_post_meta( $post_id, '_create_variation', true );
    $product_attributes   = get_post_meta( $post_id, '_product_attributes', true );
    $attribute_taxonomies = wc_get_attribute_taxonomies();

    dokan_get_template_part( 'products/product-variation', '', [
        'pro'                  => true,
        'post_id'              => $post_id,
        '_has_attribute'       => $_has_attribute,
        '_create_variations'   => $_create_variations,
        'product_attributes'   => $product_attributes,
        'attribute_taxonomies' => $attribute_taxonomies,
    ] );
}
```

**`product-variation.php`** renders the complete section:

```
┌─ Attribute & Variation Section ─────────────────────────────┐
│ <h2> "Attribute and Variation" (conditional text)           │
│                                                              │
│ ┌─ Attributes Area ────────────────────────────────────────┐ │
│ │ <ul class="dokan-attribute-option-list">                 │ │
│ │   For each existing attribute:                           │ │
│ │     dokan_get_template_part('html-product-attribute')    │ │
│ │ </ul>                                                    │ │
│ │                                                          │ │
│ │ <select id="predefined_attribute"> (taxonomy picker)     │ │
│ │ <a class="add_new_attribute"> Add attribute              │ │
│ │ <a class="dokan-save-attribute"> Save attribute          │ │
│ └──────────────────────────────────────────────────────────┘ │
│                                                              │
│ ┌─ Variations Area (show_if_variable) ─────────────────────┐ │
│ │ dokan_product_output_variations()                        │ │
│ │   → toolbar with bulk actions dropdown                   │ │
│ │   → default attribute selects                            │ │
│ │   → .dokan-variations-container (AJAX-loaded)            │ │
│ │   → Save/Cancel buttons                                  │ │
│ │   → Pagination controls                                  │ │
│ └──────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────┘
```

### 4.2 Adding a New Attribute (AJAX)

There are two paths:

**Path A — Predefined (taxonomy) attribute:**

| Property | Value |
|---|---|
| **JS Trigger** | Click `.add_new_attribute` when `#predefined_attribute` has a taxonomy selected |
| **AJAX Action** | `dokan_get_pre_attribute` |
| **Handler** | `Ajax::add_attr_predefined_attribute()` |
| **What it does** | Renders `html-product-attribute.php` with the taxonomy data, returns HTML via `wp_send_json_success()` |
| **JS callback** | Appends returned HTML to `.dokan-attribute-option-list` |

**Path B — Custom attribute:**

| Property | Value |
|---|---|
| **JS Trigger** | Click `.add_new_attribute` when `#predefined_attribute` is empty |
| **What it does** | Clones the `<script type="text/html" id="tmpl-dokan-add-attribute">` template |
| **No AJAX** | Pure client-side DOM cloning |

**Adding a new term to an existing taxonomy attribute:**

| Property | Value |
|---|---|
| **AJAX Action** | `dokan_add_new_attribute` |
| **Handler** | `Ajax::add_new_attribute()` |
| **What it does** | Calls `wp_insert_term()` and returns `{ term_id, name, slug }` |
| **JS callback** | Adds new `<option>` to the Select2 and selects it |

### 4.3 Saving Attributes (AJAX)

| Property | Value |
|---|---|
| **JS Trigger** | Click `.dokan-save-attribute` |
| **AJAX Action** | `dokan_save_attributes` |
| **Handler** | `Ajax::save_attributes()` |
| **Nonce** | None explicit (no `check_ajax_referer` — relies on logged-in user) |

**Save logic flow:**

```
1. parse_str( $_POST['data'], $data )  — deserializes the form data
2. Loop through $data['attribute_names']:
   For each attribute index $i:
     a. Read: attribute_names[$i], attribute_values[$i][], 
              attribute_visibility[$i], attribute_variation[$i],
              attribute_is_taxonomy[$i], attribute_position[$i]
     b. If taxonomy:
        - Values are term slugs → wp_set_object_terms( $post_id, $values, $taxonomy )
        - Store in attributes array with value='' (terms stored in taxonomy table)
     c. If custom:
        - Values joined with WC_DELIMITER (' | ')
        - Store in attributes array with the joined value string
3. uasort( $attributes, 'wc_product_attribute_uasort_comparison' )
4. update_post_meta( $post_id, '_product_attributes', $attributes )
```

**POST data format** (serialized from the attribute form):

```
attribute_names[0]=pa_color
attribute_values[0][]=red
attribute_values[0][]=blue
attribute_is_taxonomy[0]=1
attribute_position[0]=0
attribute_visibility[0]=1
attribute_variation[0]=1

attribute_names[1]=Custom Attr
attribute_values[1][]=Val A
attribute_values[1][]=Val B
attribute_is_taxonomy[1]=0
attribute_position[1]=1
attribute_visibility[1]=1
attribute_variation[1]=1
```

### 4.4 JS Template for New Attribute Rows

`tmpl-add-attribute.php` provides a `<script type="text/html" id="tmpl-dokan-add-attribute">` block that JS clones when adding a custom attribute without AJAX:

```html
<li class="product-attribute-list">
    <div class="dokan-product-attribute-heading">
        <span><i class="fas fa-bars"></i> <strong>Title</strong></span>
        <a href="#" class="dokan-product-remove-attribute">Remove</a>
        <a href="#" class="dokan-product-toggle-attribute">▼</a>
    </div>
    <div class="dokan-product-attribute-item dokan-clearfix dokan-hide">
        <div class="content-half-part">
            <label>Name</label>
            <input type="text" class="dokan-form-control dokan-product-attribute-name" name="" value="">
            <label><input type="checkbox" name="" value="1"> Visible on the product page</label>
            <label><input type="checkbox" name="" value="1"> Use for variations</label>
        </div>
        <div class="content-half-part">
            <label>Values</label>
            <select multiple class="dokan-select2" data-tags="true" data-token-separators="[',', '|']"></select>
        </div>
    </div>
</li>
```

JS updates the `name` attributes with the correct index after cloning.

---

## 5. How Variations Work

### 5.1 Variation Panel Output

**Function**: `dokan_product_output_variations()` in `dokan-lite/includes/Product/functions.php`

This is called from `product-variation.php` inside the `.dokan-product-variation-wrapper.show_if_variable` div.

**Output structure:**

```html
<div id="dokan-variable-product-options">
    <div id="dokan-variable-product-options-inner">
        
        <!-- If no variation attributes found -->
        <div class="dokan-alert dokan-alert-info">
            Before you can add a variation you need to add some variation attributes...
        </div>
        
        <!-- If variation attributes exist -->
        <div class="dokan-variation-top-toolbar">
            <!-- Bulk actions dropdown -->
            <select class="variation-actions">
                <option value="add_variation">Add variation</option>
                <option value="link_all_variations">Create variations from all attributes</option>
                <option value="delete_all">Delete all variations</option>
                <!-- pricing, inventory, shipping, download optgroups -->
            </select>
            <a class="do_variation_action">Go</a>
            
            <!-- Default attribute selects -->
            <div class="dokan-variations-defaults">
                <select name="default_attribute_pa_color">...</select>
            </div>
        </div>
        
        <!-- AJAX-loaded variations container -->
        <div class="dokan-variations-container wc-metaboxes"
             data-attributes='{"pa_color":{"name":"pa_color","value":"","is_variation":"1",...}}'
             data-total="12"
             data-total_pages="2"
             data-page="1">
            <!-- Variation rows loaded via AJAX -->
        </div>
        
        <!-- Toolbar: Save/Cancel + Pagination -->
        <div class="dokan-variation-action-toolbar">
            <button class="save-variation-changes">Save Variations</button>
            <button class="cancel-variation-changes">Cancel</button>
            <div class="dokan-variations-pagenav">
                <!-- First/Prev/Page Select/Next/Last -->
            </div>
        </div>
    </div>
</div>
```

**Key data attributes on `.dokan-variations-container`:**
- `data-attributes` — JSON of all product attributes (used when loading variations via AJAX)
- `data-total` — total variation count
- `data-total_pages` — total pages
- `data-page` — current page

### 5.2 Loading Variations (AJAX Pagination)

| Property | Value |
|---|---|
| **AJAX Action** | `dokan_load_variations` |
| **Handler** | `Ajax::load_variations()` |
| **Nonce** | `load-variations` |
| **Permission** | `current_user_can('dokandar')` |
| **Parameters** | `product_id`, `attributes` (JSON), `page`, `per_page` |

**Handler logic:**

```php
1. Query product_variation posts with pagination (WP_Query)
2. For each variation:
   a. Get all meta fields (_sku, _regular_price, _sale_price, _stock, etc.)
   b. Get variation attributes via wc_get_product_variation_attributes()
   c. Format prices/decimals with wc_format_localized_price/decimal
   d. Get thumbnail image URL
   e. Render html-product-variation.php template with $variation_data + $parent_data
3. die() — output is raw HTML, not JSON
```

**JS side:**

```javascript
load_variations: function( page, per_page ) {
    $.ajax({
        url: dokan.ajaxurl,
        data: {
            action:     'dokan_load_variations',
            security:   dokan.load_variations_nonce,
            product_id: $('#dokan-edit-product-id').val(),
            attributes: wrapper.data('attributes'),  // from data-attributes
            page:       page,
            per_page:   per_page
        },
        type: 'POST',
        success: function( response ) {
            wrapper.empty().append( response ).attr('data-page', page);
            $('.dokan-product-variation-wrapper').trigger('dokan_variations_loaded');
        }
    });
}
```

### 5.3 Adding a Single Variation (AJAX)

| Property | Value |
|---|---|
| **AJAX Action** | `dokan_add_variation` |
| **Handler** | `Ajax::add_variation()` |
| **Nonce** | `add-variation` |
| **Parameters** | `post_id`, `loop` (current count of variation rows) |

**Handler logic:**

```php
1. wp_insert_post() with post_type='product_variation', post_parent=$post_id
2. Build $variation_data with empty defaults
3. Build $parent_data with parent product info
4. Render html-product-variation.php
5. die() — raw HTML output
```

**JS side:**

```javascript
// Response HTML is prepended to the container
$('#dokan-variable-product-options').find('.dokan-variations-container').prepend(variation);
$('.dokan-product-variation-wrapper').trigger('dokan_variations_added', 1);
```

### 5.4 Generate All Variations ("Link All")

| Property | Value |
|---|---|
| **AJAX Action** | `dokan_link_all_variations` |
| **Handler** | `Ajax::link_all_variations()` |
| **Nonce** | `link-variations` |
| **Limit** | `WC_MAX_LINKED_VARIATIONS` (49) |

**Handler logic:**

```php
1. Get product via wc_get_product()
2. Collect all variation attributes:
   - Taxonomy → wc_get_product_terms( $post_id, $attr_name, ['fields' => 'slugs'] )
   - Custom → explode( WC_DELIMITER, $attribute['value'] )
3. Get existing variations to avoid duplicates
4. Generate cartesian product: wc_array_cartesian( $variations )
5. For each combination:
   - Skip if already exists
   - wp_insert_post( product_variation )
   - Set attribute_* post meta for each attribute
   - Set _stock_status = 'instock'
6. delete_transient( 'wc_product_children_' . $post_id )
7. echo $added (count)
```

**JS side:**

```javascript
// Shows count alert, then reloads page 1
if ( count > 0 ) {
    Dokan_Product_Variation_PageNav.go_to_page( 1, count );
}
```

### 5.5 Saving Variations (AJAX)

Dokan has **two save paths**:

#### Path A: Modern Save (via `Ajax::save_variations()`)

| Property | Value |
|---|---|
| **AJAX Action** | `dokan_save_variations` |
| **Handler** | `Ajax::save_variations()` |
| **Nonce** | `save-variations` |
| **Key detail** | **Delegates to WooCommerce's `WC_Meta_Box_Product_Data::save_variations()`** |

```php
public static function save_variations() {
    check_ajax_referer( 'save-variations', 'security' );
    
    $product_id   = absint( $_POST['product_id'] );
    $product_type = sanitize_title( stripslashes( $_POST['product_type'] ) );
    
    // Set product type if changed
    wp_set_object_terms( $product_id, $product_type, 'product_type' );
    
    // DELEGATE TO WOOCOMMERCE
    WC_Meta_Box_Product_Data::save_variations( $product_id, get_post( $product_id ) );
    
    do_action( 'dokan_ajax_save_product_variations', $product_id );
    wc_delete_product_transients( $product_id );
    die();
}
```

**This is the critical insight**: Dokan's modern save handler **directly calls WooCommerce's built-in save method**. It relies on the variation template using the exact same `name` attribute format that WC expects (`variable_regular_price[$loop]`, `variable_sku[$loop]`, etc.).

#### Path B: Legacy Save (via `dokan_save_variations()`)

In `functions-wc.php`, there's also a manual save function that loops through `$_POST['variable_sku']`, `$_POST['variable_regular_price']`, etc. and calls `update_post_meta()` for each field individually. This is the older approach used by the legacy UI.

#### JS Save Flow:

```javascript
save_changes: function( callback ) {
    var need_update = $('.variation-needs-update', wrapper);
    
    if ( 0 < need_update.length ) {
        // Only serialize fields from changed variations
        data = Dokan_Product_Variation_Ajax.get_variations_fields( need_update );
        data.action     = 'dokan_save_variations';
        data.security   = dokan.save_variations_nonce;
        data.product_id = $('#dokan-edit-product-id').val();
        data['product_type'] = $('#product_type').val();
        
        $.ajax({ url: dokan.ajaxurl, data: data, type: 'POST', ... });
    }
}

get_variations_fields: function( fields ) {
    // Uses serializeJSON plugin to build nested object from form inputs
    var data = $(':input', fields).serializeJSON();
    // Also include default attribute selects
    return data;
}
```

### 5.6 Removing a Variation (AJAX)

| Property | Value |
|---|---|
| **AJAX Action** | `dokan_remove_variation` |
| **Handler** | `Ajax::remove_variations()` |
| **Nonce** | `delete-variations` |

```php
$variation_ids = (array) $_POST['variation_ids'];
foreach ( $variation_ids as $variation_id ) {
    $variation = get_post( $variation_id );
    if ( $variation && 'product_variation' === $variation->post_type ) {
        wp_delete_post( $variation_id );
    }
}
```

### 5.7 Bulk Edit Variations (AJAX)

| Property | Value |
|---|---|
| **AJAX Action** | `dokan_bulk_edit_variations` |
| **Handler** | `Ajax::bulk_edit_variations()` |
| **Nonce** | `bulk-edit-variations` |
| **Parameters** | `product_id`, `product_type`, `bulk_action`, `data` |

**Supported bulk actions** (mapped to private static methods):

| Action | Method |
|---|---|
| `toggle_enabled` | `variation_bulk_action_toggle_enabled()` |
| `toggle_downloadable` | `variation_bulk_action_toggle_downloadable()` |
| `toggle_virtual` | `variation_bulk_action_toggle_virtual()` |
| `toggle_manage_stock` | `variation_bulk_action_toggle_manage_stock()` |
| `variable_regular_price` | `variation_bulk_action_variable_regular_price()` |
| `variable_sale_price` | `variation_bulk_action_variable_sale_price()` |
| `variable_regular_price_increase` | `variation_bulk_action_variable_regular_price_increase()` |
| `variable_regular_price_decrease` | `variation_bulk_action_variable_regular_price_decrease()` |
| `variable_sale_price_increase` | `variation_bulk_action_variable_sale_price_increase()` |
| `variable_sale_price_decrease` | `variation_bulk_action_variable_sale_price_decrease()` |
| `variable_stock` | `variation_bulk_action_variable_stock()` |
| `variable_weight` | `variation_bulk_action_variable_weight()` |
| `variable_length/width/height` | `variation_bulk_action_variable_length/width/height()` |
| `variable_download_limit` | `variation_bulk_action_variable_download_limit()` |
| `variable_download_expiry` | `variation_bulk_action_variable_download_expiry()` |
| `variable_sale_schedule` | `variation_bulk_action_variable_sale_schedule()` |
| `delete_all` | `variation_bulk_action_delete_all()` |

After bulk edit: `WC_Product_Variable::sync( $product_id )` + `wc_delete_product_transients()`.

Unknown actions fire: `do_action( 'dokan_bulk_edit_variations_default', $bulk_action, $data, $product_id, $variations )`.

---

## 6. JavaScript Architecture

### 6.1 Three JS Objects

File: `assets/src/js/product-variation.js`

```javascript
var Dokan_Product_Variation_Actions = {
    // UI behavior: checkbox toggles, expand/collapse, sorting, sale schedule
    init()
    variable_is_downloadable()  // show/hide download fields
    variable_is_virtual()       // show/hide shipping fields
    variable_manage_stock()     // show/hide stock fields
    expand_all() / close_all()
    variation_toggle_init()     // click header to expand/collapse
    variations_loaded()         // post-load: init tooltips, sortable, earning calc
    variation_added()
    set_menu_order()            // prompt for manual reorder
    variation_row_indexes()     // update menu_order after drag-drop sort
    formatMoney() / unformatMoney()  // accounting.js wrappers
};

var Dokan_Product_Variation_Ajax = {
    // CRUD operations
    init()
    check_for_changes()    // confirm before leaving with unsaved changes
    block() / unblock()    // jQuery blockUI
    initial_load()
    load_variations()      // AJAX load with pagination
    get_variations_fields() // serialize form to JSON
    save_changes()         // AJAX save (only dirty variations)
    save_variations()      // button click handler
    save_on_submit()       // intercept form submit, save variations first
    cancel_variations()    // revert to current page
    add_variation()        // AJAX add single variation
    remove_variation()     // AJAX delete
    link_all_variations()  // AJAX generate all combinations
    input_changed()        // mark variation as .variation-needs-update
    defaults_changed()
    do_variation_action()  // bulk action dispatcher (switch/case)
};

var Dokan_Product_Variation_PageNav = {
    // Pagination
    init()
    update_variations_count()
    update_single_quantity()
    set_paginav()          // rebuild page selector <select>
    check_is_enabled()
    change_classes()       // enable/disable first/prev/next/last
    set_page()
    go_to_page()
    page_selector()        // on change: load that page
    first_page() / prev_page() / next_page() / last_page()
};
```

### 6.2 Dirty Tracking & Save-on-Submit

Dokan tracks unsaved changes using the CSS class `.variation-needs-update`:

```javascript
// When any input inside a variation changes:
input_changed: function() {
    $(this).closest('.dokan-product-variation-itmes').addClass('variation-needs-update');
    $('button.cancel-variation-changes, button.save-variation-changes').removeAttr('disabled');
}

// On form submit: intercept, save variations first, then re-submit
save_on_submit: function( e ) {
    var need_update = wrapper.find('.variation-needs-update');
    if ( 0 < need_update.length ) {
        e.preventDefault();
        Dokan_Product_Variation_Ajax.save_changes( Dokan_Product_Variation_Ajax.save_on_submit_done );
    }
}

save_on_submit_done: function() {
    $('form.dokan-product-edit-form').submit();  // re-submit after save completes
}
```

**Only modified variations are serialized and sent** — not the entire form. This is efficient for products with many variations.

### 6.3 Pagination System

- **Per page**: `dokan.variations_per_page` (default 10, filterable via `dokan_product_variations_per_page`)
- **Page state**: stored in `data-page` attribute on `.dokan-variations-container`
- **Total count**: stored in `data-total` attribute
- **Navigation**: first/prev/next/last links + page `<select>` dropdown
- **On page change**: saves unsaved changes first, then loads new page via AJAX

### 6.4 Vendor Earning Calculation

Unique to Dokan: when a vendor changes a variation's price, it calculates their commission/earning in real-time:

```javascript
// On keyup of regular/sale price inputs (debounced 750ms):
$.ajax({
    url: dokan.rest.root + 'dokan/v1/commission',
    data: { product_id, amount, category_ids, context: 'seller' }
}).done( response => {
    earning_suggestion.html( formatMoney( response ) );
    // Also checks if vendor earning < 0 → disable save button
});
```

---

## 7. Variation Template (Single Row)

File: `templates/products/edit/html-product-variation.php`

Each variation renders as a collapsible card:

```
┌─ Variation Row ─────────────────────────────────────────────────────┐
│ <h3 class="variation-topbar-heading">                               │
│   #123  [pa_color: ▼ Red] [pa_size: ▼ Large]                       │
│   <input type="hidden" name="variable_post_id[$loop]">              │
│   <input type="hidden" name="variation_menu_order[$loop]">          │
│ </h3>                                                                │
│ <div class="actions"> [sort] [toggle] [remove] </div>               │
│                                                                      │
│ <div class="dokan-variable-attributes" style="display:none">         │
│   ┌─ Left Column ────────────┬─ Right Column ──────────────────┐     │
│   │ [Image Upload]           │ [SKU]                           │     │
│   │ ☑ Enabled                │ [Stock Status]                  │     │
│   │ ☑ Downloadable           │                                 │     │
│   │ ☑ Virtual                │                                 │     │
│   │ ☑ Manage stock           │                                 │     │
│   └──────────────────────────┴─────────────────────────────────┘     │
│                                                                      │
│   ┌─ Pricing ──────────────────────────────────────────────────┐     │
│   │ Regular Price ($)  [_____]   You Earn: $X.XX               │     │
│   │ Sale Price ($)     [_____]   Schedule / Cancel schedule     │     │
│   │ Sale dates: [from] [to]                                    │     │
│   └────────────────────────────────────────────────────────────┘     │
│                                                                      │
│   ┌─ Stock (if manage_stock) ──────────────────────────────────┐     │
│   │ Stock qty [___]  Allow backorders? [select]                │     │
│   │ Low stock threshold [___]                                  │     │
│   └────────────────────────────────────────────────────────────┘     │
│                                                                      │
│   ┌─ Dimensions ───────────────────────────────────────────────┐     │
│   │ Weight (kg) [___]    Dimensions L×W×H (cm) [_][_][_]      │     │
│   └────────────────────────────────────────────────────────────┘     │
│                                                                      │
│   [Shipping class] [Tax class]                                      │
│   [Description textarea]                                            │
│   [Downloadable files table] (if downloadable)                      │
│   [Download limit] [Download expiry] (if downloadable)              │
│                                                                      │
│   do_action('dokan_product_after_variable_attributes')              │
│ </div>                                                               │
└──────────────────────────────────────────────────────────────────────┘
```

**Key naming convention** (matches WooCommerce's expected format):

```
variable_post_id[$loop]           → variation ID
variable_sku[$loop]               → SKU
variable_regular_price[$loop]     → regular price
variable_sale_price[$loop]        → sale price
variable_weight[$loop]            → weight
variable_length[$loop]            → length
variable_width[$loop]             → width  
variable_height[$loop]            → height
variable_stock[$loop]             → stock quantity
variable_stock_status[$loop]      → stock status
variable_backorders[$loop]        → backorders setting
variable_tax_class[$loop]         → tax class
variable_shipping_class[$loop]    → shipping class
variable_enabled[$loop]           → enabled checkbox
variable_is_virtual[$loop]        → virtual checkbox
variable_is_downloadable[$loop]   → downloadable checkbox
variable_manage_stock[$loop]      → manage stock checkbox
variable_description[$loop]       → description
upload_image_id[$loop]            → thumbnail image ID
attribute_pa_color[$loop]         → selected attribute value
variation_menu_order[$loop]       → sort order
```

---

## 8. Nonces & Localized Data

All nonces are created in `dokan-lite/includes/Assets.php` and localized into the `dokan` JS object:

```php
wp_localize_script( 'dokan-script', 'dokan', [
    // ... other data ...
    'add_variation_nonce'        => wp_create_nonce( 'add-variation' ),
    'link_variation_nonce'       => wp_create_nonce( 'link-variations' ),
    'delete_variations_nonce'    => wp_create_nonce( 'delete-variations' ),
    'load_variations_nonce'      => wp_create_nonce( 'load-variations' ),
    'save_variations_nonce'      => wp_create_nonce( 'save-variations' ),
    'bulk_edit_variations_nonce' => wp_create_nonce( 'bulk-edit-variations' ),
    'variations_per_page'        => absint( apply_filters( 'dokan_product_variations_per_page', 10 ) ),
]);
```

**JS usage**: `dokan.load_variations_nonce`, `dokan.save_variations_nonce`, etc.

---

## 9. Key WooCommerce Reuse Points

| What Dokan Reuses | How |
|---|---|
| `WC_Meta_Box_Product_Data::save_variations()` | **Directly called** in `Ajax::save_variations()` — this is the biggest reuse |
| `wc_get_attribute_taxonomies()` | Get registered attribute taxonomies |
| `wc_attribute_label()` | Get human-readable attribute label |
| `wc_get_product_terms()` | Get terms assigned to a product for a taxonomy |
| `wc_get_text_attributes()` | Parse pipe-separated custom attribute values |
| `wc_get_product_variation_attributes()` | Get variation's selected attribute values |
| `wc_array_cartesian()` | Generate all attribute combinations for "Link All" |
| `wc_product_attribute_uasort_comparison()` | Sort attributes by position |
| `WC_Product_Variable::sync()` | Sync parent price/stock after variation changes |
| `wc_delete_product_transients()` | Clear product cache |
| `wc_format_localized_price()` / `wc_format_localized_decimal()` | Format prices for display |
| `wc_stock_amount()` | Sanitize stock values |
| `wc_product_has_unique_sku()` | Validate unique SKU |
| `wp_dropdown_categories()` | Render shipping class dropdown |
| `WC_Tax::get_tax_classes()` | Get available tax classes |
| `WC_DELIMITER` | The pipe character ` \| ` used for custom attribute values |
| jQuery BlockUI | Loading overlay via `.block()` / `.unblock()` |
| Select2/SelectWoo | Multi-select for taxonomy attribute terms |
| jQuery UI Sortable | Drag-and-drop variation reordering |
| jQuery UI Datepicker | Sale date pickers |
| `accounting.js` | Price formatting/unformatting |

**What Dokan builds custom:**
- All AJAX endpoints (own nonces, own capability checks with `dokandar` role)
- The variation pagination system (WC admin doesn't paginate the same way)
- Vendor earning calculation on price change
- Custom templates matching Dokan's UI framework
- Dirty tracking with `.variation-needs-update` class
- Image upload integration with `wp.media`

---

## 10. Data Flow Diagrams

### Saving Attributes

```
[Vendor clicks "Save attribute"]
        │
        ▼
product.js → serialize .woocommerce_attributes form
        │
        ▼
$.post( dokan_save_attributes, { post_id, data: serialized_form } )
        │
        ▼
Ajax::save_attributes()
  ├── parse_str( $_POST['data'] )
  ├── Loop attribute_names[]:
  │     ├── Taxonomy: wp_set_object_terms()
  │     └── Custom: join values with WC_DELIMITER
  ├── uasort by position
  └── update_post_meta( _product_attributes )
        │
        ▼
JS callback → reload #variable_product_options_inner via page load
             (to pick up new variation attributes)
```

### Loading Variations (Paginated)

```
[Page loads / Page nav click]
        │
        ▼
product-variation.js → load_variations( page )
        │
        ▼
$.ajax( dokan_load_variations, { product_id, attributes, page, per_page } )
        │
        ▼
Ajax::load_variations()
  ├── get_posts( product_variation, paginated )
  ├── For each variation:
  │     ├── get_post_meta( all fields )
  │     ├── wc_get_product_variation_attributes()
  │     ├── Format prices/decimals
  │     └── Render html-product-variation.php
  └── die() — raw HTML
        │
        ▼
JS callback:
  ├── wrapper.empty().append( response )
  ├── Trigger 'dokan_variations_loaded'
  │     ├── Init tooltips, sortable
  │     ├── Trigger checkbox change events
  │     └── Init earning calculation
  └── unblock()
```

### Saving Variations

```
[Vendor clicks "Save Variations" or submits form]
        │
        ▼
product-variation.js → save_changes()
  ├── Find .variation-needs-update elements
  ├── Serialize only those: $(':input', need_update).serializeJSON()
  └── Add: action, security, product_id, product_type
        │
        ▼
$.ajax( dokan_save_variations, data )
        │
        ▼
Ajax::save_variations()
  ├── wp_set_object_terms( product_type ) if changed
  ├── WC_Meta_Box_Product_Data::save_variations( $product_id, $post )
  │     └── (WooCommerce handles all field saving internally)
  ├── do_action('dokan_ajax_save_product_variations')
  └── wc_delete_product_transients()
        │
        ▼
JS callback:
  ├── Remove .variation-needs-update classes
  ├── Disable save/cancel buttons
  ├── Reload current page
  └── Trigger 'dokan_variations_saved'
```

---

## 11. Comparison: Old vs New Dokan Attribute UI

| Feature | Old (`attributes.php`) | New (`product-variation.php` + `html-product-attribute.php`) |
|---|---|---|
| **Container** | `#variants-holder` div | `.dokan-attribute-option-list` ul |
| **Attribute row** | `.inputs-box` div | `.product-attribute-list` li |
| **Collapse** | None (always expanded) | Collapsible with toggle icon |
| **Taxonomy values** | Text inputs with +/- | Select2 multi-select with all terms |
| **Custom values** | Text inputs with +/- | Select2 with `data-tags` for free text |
| **Select all/none** | Not available | Available for taxonomy attributes |
| **Add new term** | Not available | "Add new" button → `dokan_add_new_attribute` AJAX |
| **CSS framework** | Bootstrap (`btn btn-success`) | Dokan (`dokan-btn dokan-btn-default`) |
| **Save mechanism** | `dokan_save_attributes` AJAX | Same AJAX action |
| **Used for** | Older Dokan versions | Current Dokan Pro product edit form |

---

> **Key Takeaway for StoreSuite**: Dokan's most elegant shortcut is delegating `save_variations()` to `WC_Meta_Box_Product_Data::save_variations()`. This means you only need to build the UI — as long as your form fields use the same `name` attribute format (`variable_regular_price[$loop]`, etc.), WooCommerce handles all the complex save logic automatically. The AJAX handlers for load/add/remove/link are relatively straightforward `wp_insert_post` + `get_posts` + template rendering patterns that any plugin can replicate.
