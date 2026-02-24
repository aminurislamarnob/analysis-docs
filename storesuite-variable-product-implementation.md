# Implementing Variable Product & Variation Feature — StoreSuite Frontend

> **Plugin**: StoreSuite (`PluginizeLab\StoreSuite`)  
> **Scope**: Adding variable product support to the existing frontend product add/edit form  
> **Reference**: WooCommerce core + Dokan Pro patterns  
> **Generated**: February 24, 2026

---

## Table of Contents

1. [Current State Analysis](#1-current-state-analysis)
2. [Implementation Plan Overview](#2-implementation-plan-overview)
3. [Step 1 — Enable Variable Product Type](#3-step-1--enable-variable-product-type)
4. [Step 2 — Attributes Section (UI Template)](#4-step-2--attributes-section-ui-template)
5. [Step 3 — Variations Section (UI Template)](#5-step-3--variations-section-ui-template)
6. [Step 4 — Single Variation Template](#6-step-4--single-variation-template)
7. [Step 5 — JavaScript: Attributes Management](#7-step-5--javascript-attributes-management)
8. [Step 6 — JavaScript: Variations Management](#8-step-6--javascript-variations-management)
9. [Step 7 — AJAX Handlers (PHP)](#9-step-7--ajax-handlers-php)
10. [Step 8 — Save Logic (ProductManager)](#10-step-8--save-logic-productmanager)
11. [Step 9 — Conditional Show/Hide by Product Type](#11-step-9--conditional-showhide-by-product-type)
12. [Step 10 — Edit Mode: Load Existing Variation Data](#12-step-10--edit-mode-load-existing-variation-data)
13. [File Inventory & Changes Summary](#13-file-inventory--changes-summary)
14. [Data Flow Diagrams](#14-data-flow-diagrams)
15. [Database Reference](#15-database-reference)
16. [Security Checklist](#16-security-checklist)
17. [Testing Checklist](#17-testing-checklist)

---

## 1. Current State Analysis

### What StoreSuite Already Has

| Component | Status | File |
|---|---|---|
| Product type dropdown | Exists (simple only) | `templates/products/product-form.php` line 122 |
| Product type filter hook | Exists (`storesuite_product_types`) | `includes/Product/ProductHooks.php` line 28 |
| `ProductManager::create_product()` | Already handles `variable` type (prices, stock sync, default attributes) | `includes/Product/ProductManager.php` line 224 |
| Product form AJAX submission | Exists | `assets/frontend/product.js` line 111 |
| Nonce handling | Exists for add/edit | `includes/Product/ProductController.php` |
| WC product search (select2) | Already loaded for upsells/cross-sells | `templates/products/product-form.php` line 445 |

### What StoreSuite Is Missing

| Component | Needed |
|---|---|
| "Variable" option in product type dropdown | Uncomment in `ProductHooks.php` |
| Attributes section in product form | New template partial |
| Variations section in product form | New template partial |
| Single variation row template | New template partial |
| Attributes AJAX handlers (save, add, remove) | New AJAX methods |
| Variations AJAX handlers (add, save, remove, generate, load) | New AJAX methods |
| Frontend JS for attributes UI | New JS or extend `product.js` |
| Frontend JS for variations UI | New JS or extend `product.js` |
| Conditional show/hide logic per product type | New JS logic |
| Attributes + variations save logic in `ProductManager` | Extend existing method |

---

## 2. Implementation Plan Overview

```
┌──────────────────────────────────────────────────────────────────┐
│                    FILES TO MODIFY                                │
├──────────────────────────────────────────────────────────────────┤
│  includes/Product/ProductHooks.php      → Enable variable type   │
│  includes/Product/ProductController.php → Register AJAX handlers │
│  includes/Product/ProductManager.php    → Save attributes/vars   │
│  includes/Assets.php                    → Localize new JS data   │
│  templates/products/product-form.php    → Add sections           │
│  assets/frontend/product.js             → Type toggle logic      │
├──────────────────────────────────────────────────────────────────┤
│                    FILES TO CREATE                                │
├──────────────────────────────────────────────────────────────────┤
│  templates/products/product-attribute.php    → Attribute row      │
│  templates/products/product-variations.php   → Variations wrapper │
│  templates/products/product-variation.php    → Single variation   │
│  assets/frontend/product-variation.js        → Variation JS       │
└──────────────────────────────────────────────────────────────────┘
```

---

## 3. Step 1 — Enable Variable Product Type

### File: `includes/Product/ProductHooks.php`

Uncomment the `variable` type:

```php
public function set_product_types( $product_types ) {
    $product_types = array(
        'simple'   => __( 'Simple', 'storesuite' ),
        'variable' => __( 'Variable', 'storesuite' ),
    );
    return $product_types;
}
```

This will immediately show "Variable" in the product type `<select>` dropdown on the form since `product-form.php` already loops through `$product_types` at line 523.

---

## 4. Step 2 — Attributes Section (UI Template)

### New File: `templates/products/product-attribute.php`

This template renders a single attribute row (reusable for both existing and newly added attributes).

```php
<?php
/**
 * Single product attribute row template
 *
 * @var int    $i               Attribute index
 * @var string $attribute_name  Attribute name/taxonomy
 * @var array  $attribute       Attribute data array
 * @var array  $options         Attribute option values
 * @var bool   $is_taxonomy     Whether this is a taxonomy attribute
 */
if ( ! defined( 'ABSPATH' ) ) {
    exit;
}
$position = empty( $attribute['position'] ) ? 0 : absint( $attribute['position'] );
?>
<div class="storesuite-attribute-row msf-card msf-mb-12" data-index="<?php echo esc_attr( $i ); ?>">
    <div class="storesuite-attribute-header">
        <div class="row">
            <div class="col-md-5">
                <?php if ( $is_taxonomy ) : ?>
                    <strong><?php echo esc_html( wc_attribute_label( $attribute_name ) ); ?></strong>
                    <input type="hidden" name="attribute_names[<?php echo esc_attr( $i ); ?>]"
                           value="<?php echo esc_attr( $attribute_name ); ?>">
                    <input type="hidden" name="attribute_is_taxonomy[<?php echo esc_attr( $i ); ?>]" value="1">
                <?php else : ?>
                    <input type="text" class="msf-form-control" name="attribute_names[<?php echo esc_attr( $i ); ?>]"
                           value="<?php echo esc_attr( $attribute_name ); ?>"
                           placeholder="<?php esc_attr_e( 'Attribute name', 'storesuite' ); ?>">
                    <input type="hidden" name="attribute_is_taxonomy[<?php echo esc_attr( $i ); ?>]" value="0">
                <?php endif; ?>
            </div>
            <div class="col-md-5">
                <?php if ( $is_taxonomy ) : ?>
                    <select class="msf-form-control msf-select2 storesuite-attribute-values"
                            name="attribute_values[<?php echo esc_attr( $i ); ?>][]"
                            multiple data-placeholder="<?php esc_attr_e( 'Select terms', 'storesuite' ); ?>">
                        <?php
                        $all_terms = get_terms( array( 'taxonomy' => $attribute_name, 'hide_empty' => false ) );
                        if ( ! is_wp_error( $all_terms ) ) {
                            foreach ( $all_terms as $term ) {
                                $selected = in_array( $term->slug, $options, true ) ? 'selected' : '';
                                echo '<option value="' . esc_attr( $term->slug ) . '" ' . esc_attr( $selected ) . '>'
                                     . esc_html( $term->name ) . '</option>';
                            }
                        }
                        ?>
                    </select>
                <?php else : ?>
                    <input type="text" class="msf-form-control storesuite-custom-attr-values"
                           name="attribute_values_text[<?php echo esc_attr( $i ); ?>]"
                           value="<?php echo esc_attr( implode( ' | ', $options ) ); ?>"
                           placeholder="<?php esc_attr_e( 'Value 1 | Value 2 | Value 3', 'storesuite' ); ?>">
                <?php endif; ?>
            </div>
            <div class="col-md-2 text-right">
                <a href="#" class="storesuite-remove-attribute">&times;</a>
            </div>
        </div>
        <input type="hidden" name="attribute_position[<?php echo esc_attr( $i ); ?>]"
               class="attribute_position" value="<?php echo esc_attr( $position ); ?>">
    </div>
    <div class="storesuite-attribute-options">
        <label class="storesuite-checkbox-inline">
            <input type="checkbox" name="attribute_visibility[<?php echo esc_attr( $i ); ?>]"
                   value="1" <?php checked( ! empty( $attribute['is_visible'] ) ); ?>>
            <?php esc_html_e( 'Visible on product page', 'storesuite' ); ?>
        </label>
        <label class="storesuite-checkbox-inline show_if_variable">
            <input type="checkbox" name="attribute_variation[<?php echo esc_attr( $i ); ?>]"
                   value="1" <?php checked( ! empty( $attribute['is_variation'] ) ); ?>>
            <?php esc_html_e( 'Used for variations', 'storesuite' ); ?>
        </label>
    </div>
</div>
```

### Insert Attributes Section in `product-form.php`

After the Pricing card and before the Inventory card (around line 319), add an **Attributes** card. This section should appear for all product types but the "Used for variations" checkbox only shows for variable:

```php
<div class="msf-card msf-card-with-header msf-mb-24" id="storesuite-attributes-section">
    <h3 class="msf-card-title"><?php esc_html_e( 'Attributes', 'storesuite' ); ?></h3>
    <div class="msf-card-content">
        <div id="storesuite-attributes-container">
            <?php
            // Load existing attributes in edit mode
            if ( $is_edit_mode && $product ) {
                $attributes = $product->get_attributes();
                $i = 0;
                foreach ( $attributes as $attr_key => $attribute ) {
                    $is_taxonomy = $attribute->is_taxonomy();
                    $attr_name   = $attribute->get_name();
                    if ( $is_taxonomy ) {
                        $options = wp_list_pluck(
                            wp_get_post_terms( $product_id, $attr_name ),
                            'slug'
                        );
                    } else {
                        $options = $attribute->get_options();
                    }

                    $attr_data = array(
                        'name'         => $attr_name,
                        'is_visible'   => $attribute->get_visible(),
                        'is_variation' => $attribute->get_variation(),
                        'is_taxonomy'  => $is_taxonomy,
                        'position'     => $attribute->get_position(),
                    );

                    storesuite_get_template_part( 'products/product-attribute', '', array(
                        'i'              => $i,
                        'attribute_name' => $attr_name,
                        'attribute'      => $attr_data,
                        'options'        => $options,
                        'is_taxonomy'    => $is_taxonomy,
                    ) );
                    $i++;
                }
            }
            ?>
        </div>

        <div class="row msf-mt-12">
            <div class="col-md-6">
                <select class="msf-form-control" id="storesuite-add-attribute-select">
                    <option value=""><?php esc_html_e( 'Custom Attribute', 'storesuite' ); ?></option>
                    <?php
                    $attribute_taxonomies = wc_get_attribute_taxonomies();
                    foreach ( $attribute_taxonomies as $tax ) {
                        echo '<option value="' . esc_attr( wc_attribute_taxonomy_name( $tax->attribute_name ) ) . '">'
                             . esc_html( $tax->attribute_label ) . '</option>';
                    }
                    ?>
                </select>
            </div>
            <div class="col-md-6">
                <button type="button" class="my-storesuite-button my-storesuite-button-light"
                        id="storesuite-add-attribute">
                    <?php esc_html_e( 'Add Attribute', 'storesuite' ); ?>
                </button>
                <button type="button" class="my-storesuite-button"
                        id="storesuite-save-attributes">
                    <?php esc_html_e( 'Save Attributes', 'storesuite' ); ?>
                </button>
            </div>
        </div>
    </div>
</div>
```

---

## 5. Step 3 — Variations Section (UI Template)

### New File: `templates/products/product-variations.php`

This wraps the entire variations panel, shown only for variable products:

```php
<?php
if ( ! defined( 'ABSPATH' ) ) {
    exit;
}
?>
<div class="msf-card msf-card-with-header msf-mb-24 show_if_variable" id="storesuite-variations-section" style="display:none;">
    <h3 class="msf-card-title"><?php esc_html_e( 'Variations', 'storesuite' ); ?></h3>
    <div class="msf-card-content">

        <!-- Toolbar -->
        <div class="row msf-mb-12">
            <div class="col-md-4">
                <select class="msf-form-control" id="storesuite-variation-bulk-action">
                    <option value=""><?php esc_html_e( 'Bulk actions', 'storesuite' ); ?></option>
                    <option value="set_regular_prices"><?php esc_html_e( 'Set regular prices', 'storesuite' ); ?></option>
                    <option value="set_sale_prices"><?php esc_html_e( 'Set sale prices', 'storesuite' ); ?></option>
                    <option value="toggle_enabled"><?php esc_html_e( 'Toggle "Enabled"', 'storesuite' ); ?></option>
                    <option value="delete_all"><?php esc_html_e( 'Delete all variations', 'storesuite' ); ?></option>
                </select>
            </div>
            <div class="col-md-8 text-right">
                <button type="button" class="my-storesuite-button my-storesuite-button-light"
                        id="storesuite-generate-variations">
                    <?php esc_html_e( 'Generate Variations', 'storesuite' ); ?>
                </button>
                <button type="button" class="my-storesuite-button my-storesuite-button-light"
                        id="storesuite-add-variation">
                    <?php esc_html_e( 'Add Manually', 'storesuite' ); ?>
                </button>
                <button type="button" class="my-storesuite-button"
                        id="storesuite-save-variations">
                    <?php esc_html_e( 'Save Variations', 'storesuite' ); ?>
                </button>
            </div>
        </div>

        <!-- Variations container (loaded via AJAX) -->
        <div id="storesuite-variations-container">
            <p class="storesuite-no-variations">
                <?php esc_html_e( 'No variations yet. Add some attributes with "Used for variations" checked, then generate or add variations.', 'storesuite' ); ?>
            </p>
        </div>

        <!-- Pagination -->
        <div id="storesuite-variations-pagination" class="storesuite-pagination" style="display:none;"></div>
    </div>
</div>
```

---

## 6. Step 4 — Single Variation Template

### New File: `templates/products/product-variation.php`

Each variation is rendered as a collapsible card. The `$loop` index is used for array-style `name` attributes so PHP receives all variations as arrays:

```php
<?php
/**
 * Single variation row template
 *
 * @var int    $loop           Loop index
 * @var int    $variation_id   Variation post ID (0 for new)
 * @var array  $variation_data Variation meta data
 * @var array  $parent_data    Parent product data (attributes, etc.)
 * @var object $variation      WP_Post object
 */
if ( ! defined( 'ABSPATH' ) ) {
    exit;
}

$_regular_price  = isset( $variation_data['_regular_price'] ) ? $variation_data['_regular_price'] : '';
$_sale_price     = isset( $variation_data['_sale_price'] ) ? $variation_data['_sale_price'] : '';
$_sku            = isset( $variation_data['_sku'] ) ? $variation_data['_sku'] : '';
$_stock          = isset( $variation_data['_stock'] ) ? $variation_data['_stock'] : '';
$_stock_status   = isset( $variation_data['_stock_status'] ) ? $variation_data['_stock_status'] : 'instock';
$_manage_stock   = isset( $variation_data['_manage_stock'] ) ? $variation_data['_manage_stock'] : 'no';
$_weight         = isset( $variation_data['_weight'] ) ? $variation_data['_weight'] : '';
$_length         = isset( $variation_data['_length'] ) ? $variation_data['_length'] : '';
$_width          = isset( $variation_data['_width'] ) ? $variation_data['_width'] : '';
$_height         = isset( $variation_data['_height'] ) ? $variation_data['_height'] : '';
$_thumbnail_id   = isset( $variation_data['_thumbnail_id'] ) ? absint( $variation_data['_thumbnail_id'] ) : 0;
$_description    = isset( $variation_data['_variation_description'] ) ? $variation_data['_variation_description'] : '';
$enabled         = isset( $variation->post_status ) && 'publish' === $variation->post_status;
$image_url       = $_thumbnail_id ? wp_get_attachment_image_url( $_thumbnail_id, 'thumbnail' ) : wc_placeholder_img_src();
?>

<div class="storesuite-variation-row msf-card msf-mb-12" data-variation-id="<?php echo esc_attr( $variation_id ); ?>">
    <!-- Header (always visible) -->
    <div class="storesuite-variation-header" style="cursor:pointer;">
        <div class="row align-items-center">
            <div class="col-md-1">
                <img src="<?php echo esc_url( $image_url ); ?>" width="40" height="40" alt="">
            </div>
            <div class="col-md-1">
                <strong>#<?php echo esc_html( $variation_id ); ?></strong>
            </div>
            <div class="col-md-8">
                <?php
                foreach ( $parent_data['attributes'] as $attribute ) {
                    if ( empty( $attribute['is_variation'] ) ) {
                        continue;
                    }
                    $attr_key = 'attribute_' . sanitize_title( $attribute['name'] );
                    $selected = isset( $variation_data[ $attr_key ] ) ? $variation_data[ $attr_key ] : '';
                    ?>
                    <select class="msf-form-control msf-form-control-sm storesuite-variation-attr"
                            name="attribute_<?php echo esc_attr( sanitize_title( $attribute['name'] ) ); ?>[<?php echo esc_attr( $loop ); ?>]"
                            style="display:inline-block;width:auto;min-width:120px;">
                        <option value="">
                            <?php echo esc_html( sprintf( __( 'Any %s', 'storesuite' ), wc_attribute_label( $attribute['name'] ) ) ); ?>
                        </option>
                        <?php
                        if ( $attribute['is_taxonomy'] ) {
                            $terms = get_terms( array( 'taxonomy' => $attribute['name'], 'hide_empty' => false ) );
                            if ( ! is_wp_error( $terms ) ) {
                                foreach ( $terms as $term ) {
                                    echo '<option value="' . esc_attr( $term->slug ) . '" '
                                         . selected( $selected, $term->slug, false ) . '>'
                                         . esc_html( $term->name ) . '</option>';
                                }
                            }
                        } else {
                            $options = array_map( 'trim', explode( '|', $attribute['value'] ) );
                            foreach ( $options as $option ) {
                                echo '<option value="' . esc_attr( $option ) . '" '
                                     . selected( $selected, $option, false ) . '>'
                                     . esc_html( $option ) . '</option>';
                            }
                        }
                        ?>
                    </select>
                    <?php
                }
                ?>
            </div>
            <div class="col-md-2 text-right">
                <a href="#" class="storesuite-remove-variation" data-variation-id="<?php echo esc_attr( $variation_id ); ?>">
                    <?php esc_html_e( 'Remove', 'storesuite' ); ?>
                </a>
                <a href="#" class="storesuite-toggle-variation">&#9660;</a>
            </div>
        </div>
        <input type="hidden" name="variable_post_id[<?php echo esc_attr( $loop ); ?>]"
               value="<?php echo esc_attr( $variation_id ); ?>">
        <input type="hidden" name="variation_menu_order[<?php echo esc_attr( $loop ); ?>]"
               value="<?php echo isset( $variation_data['menu_order'] ) ? esc_attr( $variation_data['menu_order'] ) : 0; ?>">
    </div>

    <!-- Body (collapsed by default) -->
    <div class="storesuite-variation-body" style="display:none;">
        <div class="msf-card-content">
            <div class="row">
                <!-- Image -->
                <div class="col-md-2">
                    <div class="storesuite-variation-image">
                        <a href="#" class="storesuite-variation-upload-image">
                            <img src="<?php echo esc_url( $image_url ); ?>" width="80" height="80" alt="">
                        </a>
                        <input type="hidden" name="upload_image_id[<?php echo esc_attr( $loop ); ?>]"
                               class="upload_image_id" value="<?php echo esc_attr( $_thumbnail_id ); ?>">
                    </div>
                    <label class="storesuite-checkbox-inline">
                        <input type="checkbox" name="variable_enabled[<?php echo esc_attr( $loop ); ?>]"
                               value="1" <?php checked( $enabled ); ?>>
                        <?php esc_html_e( 'Enabled', 'storesuite' ); ?>
                    </label>
                </div>

                <!-- Pricing + SKU + Stock -->
                <div class="col-md-10">
                    <div class="row">
                        <!-- SKU -->
                        <div class="col-md-4">
                            <div class="msf-form-group">
                                <label><?php esc_html_e( 'SKU', 'storesuite' ); ?></label>
                                <input type="text" class="msf-form-control"
                                       name="variable_sku[<?php echo esc_attr( $loop ); ?>]"
                                       value="<?php echo esc_attr( $_sku ); ?>">
                            </div>
                        </div>
                        <!-- Regular Price -->
                        <div class="col-md-4">
                            <div class="msf-form-group">
                                <label><?php echo esc_html__( 'Regular Price', 'storesuite' ) . ' (' . get_woocommerce_currency_symbol() . ')'; ?></label>
                                <input type="text" class="msf-form-control"
                                       name="variable_regular_price[<?php echo esc_attr( $loop ); ?>]"
                                       value="<?php echo esc_attr( $_regular_price ); ?>">
                            </div>
                        </div>
                        <!-- Sale Price -->
                        <div class="col-md-4">
                            <div class="msf-form-group">
                                <label><?php esc_html_e( 'Sale Price', 'storesuite' ); ?></label>
                                <input type="text" class="msf-form-control"
                                       name="variable_sale_price[<?php echo esc_attr( $loop ); ?>]"
                                       value="<?php echo esc_attr( $_sale_price ); ?>">
                            </div>
                        </div>
                    </div>

                    <div class="row">
                        <!-- Stock Status -->
                        <div class="col-md-4">
                            <div class="msf-form-group">
                                <label><?php esc_html_e( 'Stock Status', 'storesuite' ); ?></label>
                                <select class="msf-form-control"
                                        name="variable_stock_status[<?php echo esc_attr( $loop ); ?>]">
                                    <?php foreach ( wc_get_product_stock_status_options() as $key => $value ) : ?>
                                        <option value="<?php echo esc_attr( $key ); ?>"
                                            <?php selected( $_stock_status, $key ); ?>>
                                            <?php echo esc_html( $value ); ?>
                                        </option>
                                    <?php endforeach; ?>
                                </select>
                            </div>
                        </div>
                        <!-- Stock Qty -->
                        <div class="col-md-4">
                            <div class="msf-form-group">
                                <label><?php esc_html_e( 'Stock Qty', 'storesuite' ); ?></label>
                                <input type="number" class="msf-form-control"
                                       name="variable_stock[<?php echo esc_attr( $loop ); ?>]"
                                       value="<?php echo esc_attr( $_stock ); ?>" step="any">
                            </div>
                        </div>
                        <!-- Manage Stock -->
                        <div class="col-md-4">
                            <div class="msf-form-group msf-form-switch" style="margin-top:28px;">
                                <input type="checkbox"
                                       name="variable_manage_stock[<?php echo esc_attr( $loop ); ?>]"
                                       value="yes" <?php checked( $_manage_stock, 'yes' ); ?>>
                                <label><?php esc_html_e( 'Manage stock?', 'storesuite' ); ?></label>
                            </div>
                        </div>
                    </div>

                    <div class="row">
                        <!-- Weight -->
                        <div class="col-md-3">
                            <div class="msf-form-group">
                                <label><?php echo esc_html__( 'Weight', 'storesuite' ) . ' (' . esc_html( get_option( 'woocommerce_weight_unit' ) ) . ')'; ?></label>
                                <input type="text" class="msf-form-control"
                                       name="variable_weight[<?php echo esc_attr( $loop ); ?>]"
                                       value="<?php echo esc_attr( $_weight ); ?>">
                            </div>
                        </div>
                        <!-- Dimensions -->
                        <div class="col-md-3">
                            <div class="msf-form-group">
                                <label><?php esc_html_e( 'Length', 'storesuite' ); ?></label>
                                <input type="text" class="msf-form-control"
                                       name="variable_length[<?php echo esc_attr( $loop ); ?>]"
                                       value="<?php echo esc_attr( $_length ); ?>">
                            </div>
                        </div>
                        <div class="col-md-3">
                            <div class="msf-form-group">
                                <label><?php esc_html_e( 'Width', 'storesuite' ); ?></label>
                                <input type="text" class="msf-form-control"
                                       name="variable_width[<?php echo esc_attr( $loop ); ?>]"
                                       value="<?php echo esc_attr( $_width ); ?>">
                            </div>
                        </div>
                        <div class="col-md-3">
                            <div class="msf-form-group">
                                <label><?php esc_html_e( 'Height', 'storesuite' ); ?></label>
                                <input type="text" class="msf-form-control"
                                       name="variable_height[<?php echo esc_attr( $loop ); ?>]"
                                       value="<?php echo esc_attr( $_height ); ?>">
                            </div>
                        </div>
                    </div>

                    <div class="row">
                        <div class="col-md-12">
                            <div class="msf-form-group">
                                <label><?php esc_html_e( 'Variation Description', 'storesuite' ); ?></label>
                                <textarea class="msf-form-control"
                                          name="variable_description[<?php echo esc_attr( $loop ); ?>]"
                                          rows="2"><?php echo esc_textarea( $_description ); ?></textarea>
                            </div>
                        </div>
                    </div>
                </div>
            </div>
        </div>
    </div>
</div>
```

---

## 7. Step 5 — JavaScript: Attributes Management

Add the following logic to `assets/frontend/product.js` (or a separate file like `product-variation.js`):

```javascript
var StoreSuiteAttributes = {
    attributeIndex: 0, // Track next available index

    init: function() {
        this.attributeIndex = $('#storesuite-attributes-container .storesuite-attribute-row').length;
        this.bindEvents();
    },

    bindEvents: function() {
        $(document).on('click', '#storesuite-add-attribute', this.addAttribute.bind(this));
        $(document).on('click', '.storesuite-remove-attribute', this.removeAttribute.bind(this));
        $(document).on('click', '#storesuite-save-attributes', this.saveAttributes.bind(this));
    },

    addAttribute: function(e) {
        e.preventDefault();
        var taxonomy = $('#storesuite-add-attribute-select').val();
        var idx = this.attributeIndex++;

        $.ajax({
            url: StoreSuite_Product.ajax_url,
            type: 'POST',
            data: {
                action: 'storesuite_add_attribute_row',
                taxonomy: taxonomy,
                index: idx,
                _wpnonce: StoreSuite_Product.attribute_nonce
            },
            success: function(response) {
                if (response.success) {
                    $('#storesuite-attributes-container').append(response.data.html);
                    // Re-init select2 on new elements
                    $('#storesuite-attributes-container .msf-select2').not('.enhanced')
                        .selectWoo({ width: '100%' }).addClass('enhanced');
                }
            }
        });
    },

    removeAttribute: function(e) {
        e.preventDefault();
        $(e.target).closest('.storesuite-attribute-row').slideUp(200, function() {
            $(this).remove();
        });
    },

    saveAttributes: function(e) {
        e.preventDefault();
        var $form = $('#msfc-add-product');
        var productId = $form.find('input[name="product_id"]').val() || 0;
        var data = $form.serialize();

        $.ajax({
            url: StoreSuite_Product.ajax_url,
            type: 'POST',
            data: data + '&action=storesuite_save_attributes&product_id=' + productId +
                  '&_wpnonce=' + StoreSuite_Product.attribute_nonce,
            success: function(response) {
                if (response.success) {
                    // Refresh variations section if product type is variable
                    if ($('#post_type').val() === 'variable') {
                        StoreSuiteVariations.loadVariations();
                    }
                }
            }
        });
    }
};
```

---

## 8. Step 6 — JavaScript: Variations Management

```javascript
var StoreSuiteVariations = {
    currentPage: 1,
    perPage: 10,

    init: function() {
        this.bindEvents();
        this.toggleSections();
    },

    bindEvents: function() {
        $(document).on('change', '#post_type', this.toggleSections.bind(this));
        $(document).on('click', '#storesuite-generate-variations', this.generateVariations.bind(this));
        $(document).on('click', '#storesuite-add-variation', this.addVariation.bind(this));
        $(document).on('click', '#storesuite-save-variations', this.saveVariations.bind(this));
        $(document).on('click', '.storesuite-remove-variation', this.removeVariation.bind(this));
        $(document).on('click', '.storesuite-toggle-variation', this.toggleVariation.bind(this));
        $(document).on('click', '.storesuite-variation-header', this.toggleVariation.bind(this));
        $(document).on('click', '.storesuite-variation-upload-image', this.uploadImage.bind(this));
    },

    toggleSections: function() {
        var type = $('#post_type').val();
        if (type === 'variable') {
            $('.show_if_variable').show();
            // Hide simple-only pricing/inventory at parent level
            // (variable products have prices per-variation)
        } else {
            $('.show_if_variable').hide();
        }
    },

    loadVariations: function(page) {
        page = page || 1;
        var productId = $('input[name="product_id"]').val() || 0;
        if (!productId) return;

        $.ajax({
            url: StoreSuite_Product.ajax_url,
            type: 'POST',
            data: {
                action: 'storesuite_load_variations',
                product_id: productId,
                page: page,
                per_page: this.perPage,
                _wpnonce: StoreSuite_Product.variation_nonce
            },
            success: function(response) {
                if (response.success) {
                    $('#storesuite-variations-container').html(response.data.html);
                    // Show pagination if needed
                    if (response.data.total_pages > 1) {
                        // Build pagination
                    }
                }
            }
        });
    },

    generateVariations: function(e) {
        e.preventDefault();
        var productId = $('input[name="product_id"]').val() || 0;

        if (!confirm(StoreSuite_Product.i18n.confirm_generate_variations ||
                     'Generate all variations? This will create variations for all attribute combinations.')) {
            return;
        }

        window.StoreSuite.msfcLoader.block($('.my-storesuite-wrapper'));

        $.ajax({
            url: StoreSuite_Product.ajax_url,
            type: 'POST',
            data: {
                action: 'storesuite_generate_variations',
                product_id: productId,
                _wpnonce: StoreSuite_Product.variation_nonce
            },
            success: function(response) {
                window.StoreSuite.msfcLoader.unblock($('.my-storesuite-wrapper'));
                if (response.success) {
                    StoreSuiteVariations.loadVariations();
                    Swal.fire({
                        icon: 'success',
                        title: StoreSuite_Product.i18n.success_title,
                        text: response.data.message
                    });
                }
            }
        });
    },

    addVariation: function(e) {
        e.preventDefault();
        var productId = $('input[name="product_id"]').val() || 0;
        var loop = $('#storesuite-variations-container .storesuite-variation-row').length;

        $.ajax({
            url: StoreSuite_Product.ajax_url,
            type: 'POST',
            data: {
                action: 'storesuite_add_variation',
                product_id: productId,
                loop: loop,
                _wpnonce: StoreSuite_Product.variation_nonce
            },
            success: function(response) {
                if (response.success) {
                    $('#storesuite-variations-container .storesuite-no-variations').remove();
                    $('#storesuite-variations-container').append(response.data.html);
                }
            }
        });
    },

    saveVariations: function(e) {
        e.preventDefault();
        var $form = $('#msfc-add-product');
        var formData = new FormData($form[0]);
        formData.append('action', 'storesuite_save_variations');
        formData.append('_wpnonce', StoreSuite_Product.variation_nonce);

        window.StoreSuite.msfcLoader.block($('.my-storesuite-wrapper'));

        $.ajax({
            url: StoreSuite_Product.ajax_url,
            type: 'POST',
            data: formData,
            processData: false,
            contentType: false,
            success: function(response) {
                window.StoreSuite.msfcLoader.unblock($('.my-storesuite-wrapper'));
                if (response.success) {
                    Swal.fire({
                        icon: 'success',
                        title: StoreSuite_Product.i18n.success_title,
                        text: response.data.message
                    });
                }
            }
        });
    },

    removeVariation: function(e) {
        e.preventDefault();
        e.stopPropagation();
        var $row = $(e.target).closest('.storesuite-variation-row');
        var variationId = $row.data('variation-id');

        if (!confirm('Remove this variation?')) return;

        if (variationId) {
            $.ajax({
                url: StoreSuite_Product.ajax_url,
                type: 'POST',
                data: {
                    action: 'storesuite_remove_variation',
                    variation_id: variationId,
                    _wpnonce: StoreSuite_Product.variation_nonce
                }
            });
        }
        $row.slideUp(200, function() { $(this).remove(); });
    },

    toggleVariation: function(e) {
        e.preventDefault();
        var $row = $(e.target).closest('.storesuite-variation-row');
        $row.find('.storesuite-variation-body').slideToggle(200);
    },

    uploadImage: function(e) {
        e.preventDefault();
        var $link = $(e.target).closest('.storesuite-variation-upload-image');
        var $row = $link.closest('.storesuite-variation-row');
        var frame = wp.media({ title: 'Variation Image', multiple: false });

        frame.on('select', function() {
            var attachment = frame.state().get('selection').first().toJSON();
            $row.find('.upload_image_id').val(attachment.id);
            $row.find('.storesuite-variation-upload-image img').attr('src', attachment.sizes.thumbnail.url);
            $row.find('.storesuite-variation-header img').attr('src', attachment.sizes.thumbnail.url);
        });

        frame.open();
    }
};

// Initialize on document ready
$(document).ready(function() {
    StoreSuiteAttributes.init();
    StoreSuiteVariations.init();
});
```

---

## 9. Step 7 — AJAX Handlers (PHP)

### File: `includes/Product/ProductController.php`

Register new AJAX actions in `__construct()`:

```php
public function __construct() {
    // ... existing hooks ...

    // Attribute AJAX
    add_action( 'wp_ajax_storesuite_add_attribute_row', array( $this, 'handle_add_attribute_row' ) );
    add_action( 'wp_ajax_storesuite_save_attributes', array( $this, 'handle_save_attributes' ) );

    // Variation AJAX
    add_action( 'wp_ajax_storesuite_load_variations', array( $this, 'handle_load_variations' ) );
    add_action( 'wp_ajax_storesuite_add_variation', array( $this, 'handle_add_variation' ) );
    add_action( 'wp_ajax_storesuite_generate_variations', array( $this, 'handle_generate_variations' ) );
    add_action( 'wp_ajax_storesuite_save_variations', array( $this, 'handle_save_variations' ) );
    add_action( 'wp_ajax_storesuite_remove_variation', array( $this, 'handle_remove_variation' ) );
}
```

### Example: `handle_save_attributes()`

```php
public function handle_save_attributes() {
    check_ajax_referer( 'storesuite_attribute_nonce', '_wpnonce' );

    if ( ! current_user_can( 'manage_woocommerce' ) ) {
        wp_send_json_error( array( 'error' => __( 'Permission denied.', 'storesuite' ) ) );
    }

    $product_id      = isset( $_POST['product_id'] ) ? absint( $_POST['product_id'] ) : 0;
    $attribute_names  = isset( $_POST['attribute_names'] ) ? wc_clean( wp_unslash( $_POST['attribute_names'] ) ) : array();
    $attribute_values = isset( $_POST['attribute_values'] ) ? wc_clean( wp_unslash( $_POST['attribute_values'] ) ) : array();
    $attribute_text   = isset( $_POST['attribute_values_text'] ) ? wc_clean( wp_unslash( $_POST['attribute_values_text'] ) ) : array();
    $attribute_vis    = isset( $_POST['attribute_visibility'] ) ? wc_clean( wp_unslash( $_POST['attribute_visibility'] ) ) : array();
    $attribute_var    = isset( $_POST['attribute_variation'] ) ? wc_clean( wp_unslash( $_POST['attribute_variation'] ) ) : array();
    $attribute_tax    = isset( $_POST['attribute_is_taxonomy'] ) ? wc_clean( wp_unslash( $_POST['attribute_is_taxonomy'] ) ) : array();
    $attribute_pos    = isset( $_POST['attribute_position'] ) ? wc_clean( wp_unslash( $_POST['attribute_position'] ) ) : array();

    if ( ! $product_id ) {
        wp_send_json_error( array( 'error' => __( 'Save the product first.', 'storesuite' ) ) );
    }

    $product    = wc_get_product( $product_id );
    $attributes = array();

    foreach ( $attribute_names as $i => $name ) {
        if ( empty( $name ) ) {
            continue;
        }

        $is_taxonomy = ! empty( $attribute_tax[ $i ] );
        $attribute   = new \WC_Product_Attribute();

        if ( $is_taxonomy ) {
            $attribute->set_id( wc_attribute_taxonomy_id_by_name( $name ) );
            $attribute->set_name( $name );

            $values = isset( $attribute_values[ $i ] ) ? $attribute_values[ $i ] : array();
            $term_ids = array();
            foreach ( $values as $slug ) {
                $term = get_term_by( 'slug', $slug, $name );
                if ( $term ) {
                    $term_ids[] = $term->term_id;
                }
            }
            $attribute->set_options( $term_ids );
        } else {
            $attribute->set_id( 0 );
            $attribute->set_name( $name );
            $values = isset( $attribute_text[ $i ] ) ? array_map( 'trim', explode( '|', $attribute_text[ $i ] ) ) : array();
            $attribute->set_options( $values );
        }

        $attribute->set_position( isset( $attribute_pos[ $i ] ) ? absint( $attribute_pos[ $i ] ) : 0 );
        $attribute->set_visible( isset( $attribute_vis[ $i ] ) );
        $attribute->set_variation( isset( $attribute_var[ $i ] ) );

        $attributes[] = $attribute;
    }

    $product->set_attributes( $attributes );
    $product->save();

    wp_send_json_success( array( 'message' => __( 'Attributes saved.', 'storesuite' ) ) );
}
```

### Example: `handle_generate_variations()`

```php
public function handle_generate_variations() {
    check_ajax_referer( 'storesuite_variation_nonce', '_wpnonce' );

    if ( ! current_user_can( 'manage_woocommerce' ) ) {
        wp_send_json_error( array( 'error' => __( 'Permission denied.', 'storesuite' ) ) );
    }

    $product_id = isset( $_POST['product_id'] ) ? absint( $_POST['product_id'] ) : 0;
    $product    = wc_get_product( $product_id );

    if ( ! $product || ! $product->is_type( 'variable' ) ) {
        wp_send_json_error( array( 'error' => __( 'Invalid product.', 'storesuite' ) ) );
    }

    // Get variation attributes
    $attributes = $product->get_attributes();
    $variation_attributes = array();

    foreach ( $attributes as $attribute ) {
        if ( ! $attribute->get_variation() ) {
            continue;
        }

        $attr_name = $attribute->get_name();
        if ( $attribute->is_taxonomy() ) {
            $terms = wp_get_post_terms( $product_id, $attr_name, array( 'fields' => 'slugs' ) );
            $variation_attributes[ $attr_name ] = $terms;
        } else {
            $variation_attributes[ $attr_name ] = $attribute->get_options();
        }
    }

    if ( empty( $variation_attributes ) ) {
        wp_send_json_error( array( 'error' => __( 'No variation attributes found.', 'storesuite' ) ) );
    }

    // Generate all combinations
    $combinations = array( array() );
    foreach ( $variation_attributes as $attr_name => $values ) {
        $new_combinations = array();
        foreach ( $combinations as $combo ) {
            foreach ( $values as $value ) {
                $new_combinations[] = array_merge( $combo, array( $attr_name => $value ) );
            }
        }
        $combinations = $new_combinations;
    }

    // Get existing variations to avoid duplicates
    $existing = array();
    foreach ( $product->get_children() as $child_id ) {
        $child = wc_get_product( $child_id );
        if ( $child ) {
            $existing[] = $child->get_attributes();
        }
    }

    $created = 0;
    foreach ( $combinations as $combo ) {
        // Check if this combination already exists
        $attrs = array();
        foreach ( $combo as $name => $value ) {
            $key = sanitize_title( $name );
            $attrs[ $key ] = $value;
        }

        $already_exists = false;
        foreach ( $existing as $ex_attrs ) {
            if ( $ex_attrs === $attrs ) {
                $already_exists = true;
                break;
            }
        }

        if ( $already_exists ) {
            continue;
        }

        $variation = new \WC_Product_Variation();
        $variation->set_parent_id( $product_id );
        $variation->set_attributes( $attrs );
        $variation->set_status( 'publish' );
        $variation->save();
        $created++;
    }

    wp_send_json_success( array(
        'message' => sprintf( __( '%d variations created.', 'storesuite' ), $created ),
        'count'   => $created,
    ) );
}
```

### Example: `handle_save_variations()`

```php
public function handle_save_variations() {
    check_ajax_referer( 'storesuite_variation_nonce', '_wpnonce' );

    if ( ! current_user_can( 'manage_woocommerce' ) ) {
        wp_send_json_error( array( 'error' => __( 'Permission denied.', 'storesuite' ) ) );
    }

    $product_id     = isset( $_POST['product_id'] ) ? absint( $_POST['product_id'] ) : 0;
    $variation_ids  = isset( $_POST['variable_post_id'] ) ? array_map( 'absint', wp_unslash( $_POST['variable_post_id'] ) ) : array();
    $max_loop       = max( array_keys( $variation_ids ) );

    for ( $i = 0; $i <= $max_loop; $i++ ) {
        if ( ! isset( $variation_ids[ $i ] ) ) {
            continue;
        }

        $variation_id = absint( $variation_ids[ $i ] );
        $variation    = wc_get_product( $variation_id );

        if ( ! $variation || ! $variation->is_type( 'variation' ) ) {
            continue;
        }

        // Enabled
        $status = isset( $_POST['variable_enabled'][ $i ] ) ? 'publish' : 'private';
        $variation->set_status( $status );

        // SKU
        if ( isset( $_POST['variable_sku'][ $i ] ) ) {
            $variation->set_sku( wc_clean( wp_unslash( $_POST['variable_sku'][ $i ] ) ) );
        }

        // Pricing
        if ( isset( $_POST['variable_regular_price'][ $i ] ) ) {
            $variation->set_regular_price( wc_format_decimal( wp_unslash( $_POST['variable_regular_price'][ $i ] ) ) );
        }
        if ( isset( $_POST['variable_sale_price'][ $i ] ) ) {
            $variation->set_sale_price( wc_format_decimal( wp_unslash( $_POST['variable_sale_price'][ $i ] ) ) );
        }

        // Stock
        $manage_stock = isset( $_POST['variable_manage_stock'][ $i ] ) ? 'yes' : 'no';
        $variation->set_manage_stock( $manage_stock );

        if ( 'yes' === $manage_stock && isset( $_POST['variable_stock'][ $i ] ) ) {
            $variation->set_stock_quantity( wc_stock_amount( wp_unslash( $_POST['variable_stock'][ $i ] ) ) );
        }
        if ( isset( $_POST['variable_stock_status'][ $i ] ) ) {
            $variation->set_stock_status( wc_clean( wp_unslash( $_POST['variable_stock_status'][ $i ] ) ) );
        }

        // Shipping
        if ( isset( $_POST['variable_weight'][ $i ] ) ) {
            $variation->set_weight( wc_clean( wp_unslash( $_POST['variable_weight'][ $i ] ) ) );
        }
        if ( isset( $_POST['variable_length'][ $i ] ) ) {
            $variation->set_length( wc_clean( wp_unslash( $_POST['variable_length'][ $i ] ) ) );
        }
        if ( isset( $_POST['variable_width'][ $i ] ) ) {
            $variation->set_width( wc_clean( wp_unslash( $_POST['variable_width'][ $i ] ) ) );
        }
        if ( isset( $_POST['variable_height'][ $i ] ) ) {
            $variation->set_height( wc_clean( wp_unslash( $_POST['variable_height'][ $i ] ) ) );
        }

        // Image
        if ( isset( $_POST['upload_image_id'][ $i ] ) ) {
            $variation->set_image_id( absint( $_POST['upload_image_id'][ $i ] ) );
        }

        // Description
        if ( isset( $_POST['variable_description'][ $i ] ) ) {
            $variation->set_description( wp_kses_post( wp_unslash( $_POST['variable_description'][ $i ] ) ) );
        }

        // Variation attributes
        $parent = wc_get_product( $product_id );
        $parent_attributes = $parent->get_attributes();
        $variation_attrs = array();

        foreach ( $parent_attributes as $attribute ) {
            if ( ! $attribute->get_variation() ) {
                continue;
            }
            $attr_key = 'attribute_' . sanitize_title( $attribute->get_name() );
            if ( isset( $_POST[ $attr_key ][ $i ] ) ) {
                $variation_attrs[ sanitize_title( $attribute->get_name() ) ] = wc_clean( wp_unslash( $_POST[ $attr_key ][ $i ] ) );
            }
        }
        $variation->set_attributes( $variation_attrs );

        $variation->save();

        do_action( 'storesuite_save_product_variation', $variation_id, $i );
    }

    // Sync parent product
    $parent = wc_get_product( $product_id );
    if ( $parent ) {
        \WC_Product_Variable::sync( $parent );
    }

    wp_send_json_success( array( 'message' => __( 'Variations saved.', 'storesuite' ) ) );
}
```

### Example: `handle_load_variations()`

```php
public function handle_load_variations() {
    check_ajax_referer( 'storesuite_variation_nonce', '_wpnonce' );

    $product_id = isset( $_POST['product_id'] ) ? absint( $_POST['product_id'] ) : 0;
    $page       = isset( $_POST['page'] ) ? absint( $_POST['page'] ) : 1;
    $per_page   = isset( $_POST['per_page'] ) ? absint( $_POST['per_page'] ) : 10;
    $product    = wc_get_product( $product_id );

    if ( ! $product || ! $product->is_type( 'variable' ) ) {
        wp_send_json_error();
    }

    $children    = $product->get_children();
    $total       = count( $children );
    $total_pages = ceil( $total / $per_page );
    $offset      = ( $page - 1 ) * $per_page;
    $page_ids    = array_slice( $children, $offset, $per_page );

    // Prepare parent data for the template
    $parent_data = array(
        'id'         => $product_id,
        'attributes' => array(),
    );

    $raw_attrs = maybe_unserialize( get_post_meta( $product_id, '_product_attributes', true ) );
    if ( is_array( $raw_attrs ) ) {
        $parent_data['attributes'] = $raw_attrs;
    }

    ob_start();
    $loop = $offset;
    foreach ( $page_ids as $variation_id ) {
        $variation      = get_post( $variation_id );
        $variation_data = get_post_meta( $variation_id );

        // Flatten meta arrays
        $flat_data = array();
        foreach ( $variation_data as $key => $values ) {
            $flat_data[ $key ] = $values[0];
        }
        $flat_data['menu_order'] = $variation->menu_order;

        storesuite_get_template_part( 'products/product-variation', '', array(
            'loop'           => $loop,
            'variation_id'   => $variation_id,
            'variation_data' => $flat_data,
            'parent_data'    => $parent_data,
            'variation'      => $variation,
        ) );
        $loop++;
    }
    $html = ob_get_clean();

    wp_send_json_success( array(
        'html'        => $html,
        'total'       => $total,
        'total_pages' => $total_pages,
        'page'        => $page,
    ) );
}
```

---

## 10. Step 8 — Save Logic (ProductManager)

### Changes to `ProductManager::storesuite_save_product()`

The `create_product()` method already handles `type => 'variable'` correctly (clears parent prices, syncs default attributes). You need to add:

1. **Before creating the product**, if `type === 'variable'`, pass attributes from POST:

```php
// In storesuite_save_product(), before calling create_product():
if ( 'variable' === $post_data['type'] && ! empty( $data['attribute_names'] ) ) {
    $attributes = $this->prepare_attributes_from_post( $data );
    $post_data['attributes'] = $attributes;
}
```

2. **Add helper method**:

```php
protected function prepare_attributes_from_post( $data ) {
    $attribute_names  = $data['attribute_names'] ?? array();
    $attribute_values = $data['attribute_values'] ?? array();
    $attribute_text   = $data['attribute_values_text'] ?? array();
    $attribute_vis    = $data['attribute_visibility'] ?? array();
    $attribute_var    = $data['attribute_variation'] ?? array();
    $attribute_tax    = $data['attribute_is_taxonomy'] ?? array();
    $attribute_pos    = $data['attribute_position'] ?? array();

    $attributes = array();
    foreach ( $attribute_names as $i => $name ) {
        if ( empty( $name ) ) continue;

        $attribute = new \WC_Product_Attribute();
        $is_taxonomy = ! empty( $attribute_tax[ $i ] );

        if ( $is_taxonomy ) {
            $attribute->set_id( wc_attribute_taxonomy_id_by_name( $name ) );
            $attribute->set_name( $name );
            $values = isset( $attribute_values[ $i ] ) ? (array) $attribute_values[ $i ] : array();
            $term_ids = array();
            foreach ( $values as $slug ) {
                $term = get_term_by( 'slug', $slug, $name );
                if ( $term ) $term_ids[] = $term->term_id;
            }
            $attribute->set_options( $term_ids );
        } else {
            $attribute->set_id( 0 );
            $attribute->set_name( $name );
            $values = isset( $attribute_text[ $i ] ) ? array_map( 'trim', explode( '|', $attribute_text[ $i ] ) ) : array();
            $attribute->set_options( $values );
        }

        $attribute->set_position( isset( $attribute_pos[ $i ] ) ? absint( $attribute_pos[ $i ] ) : $i );
        $attribute->set_visible( isset( $attribute_vis[ $i ] ) );
        $attribute->set_variation( isset( $attribute_var[ $i ] ) );

        $attributes[] = $attribute;
    }

    return $attributes;
}
```

---

## 11. Step 9 — Conditional Show/Hide by Product Type

When the user switches product type in the dropdown, the form must adapt:

| Section | Simple | Variable |
|---|---|---|
| Pricing (parent level) | Show | **Hide** (prices are per-variation) |
| Inventory (parent level) | Show | Show (optional stock management) |
| Attributes | Show (optional) | Show |
| "Used for variations" checkbox | **Hide** | Show |
| Variations section | **Hide** | Show |

JavaScript logic (add to `product.js` or the new variation JS):

```javascript
$(document).on('change', '#post_type', function() {
    var type = $(this).val();

    // Pricing card — hide for variable
    var $pricingCard = $('h3.msf-card-title').filter(function() {
        return $(this).text().trim() === 'Pricing';
    }).closest('.msf-card');

    if (type === 'variable') {
        $pricingCard.slideUp();
        $('.show_if_variable').slideDown();
        $('.show_if_simple').slideUp();
    } else {
        $pricingCard.slideDown();
        $('.show_if_variable').slideUp();
        $('.show_if_simple').slideDown();
    }
});

// Trigger on page load for edit mode
$('#post_type').trigger('change');
```

---

## 12. Step 10 — Edit Mode: Load Existing Variation Data

When editing a variable product, variations need to be loaded. In the variations template, on page load:

```javascript
$(document).ready(function() {
    var productId = $('input[name="product_id"]').val();
    var productType = $('#post_type').val();

    if (productId && productType === 'variable') {
        StoreSuiteVariations.loadVariations(1);
    }
});
```

### Localized Script Data (Assets.php)

Add nonces and i18n strings in `Assets.php` `enqueue_front_scripts()`:

```php
$product_script_data = array(
    // ... existing data ...
    'attribute_nonce'  => wp_create_nonce( 'storesuite_attribute_nonce' ),
    'variation_nonce'  => wp_create_nonce( 'storesuite_variation_nonce' ),
    'i18n' => array(
        // ... existing i18n ...
        'confirm_generate_variations' => __( 'Generate all variations? This will create a variation for every combination of attributes.', 'storesuite' ),
        'variations_saved'            => __( 'Variations saved successfully.', 'storesuite' ),
        'attributes_saved'            => __( 'Attributes saved successfully.', 'storesuite' ),
        'remove_variation_confirm'    => __( 'Are you sure you want to remove this variation?', 'storesuite' ),
    ),
);
```

---

## 13. File Inventory & Changes Summary

### Files to Modify

| File | Change |
|---|---|
| `includes/Product/ProductHooks.php` | Uncomment `'variable'` in `set_product_types()` |
| `includes/Product/ProductController.php` | Add 5 new AJAX handler registrations + methods |
| `includes/Product/ProductManager.php` | Add `prepare_attributes_from_post()` method, modify `storesuite_save_product()` to pass attributes |
| `includes/Assets.php` | Add `attribute_nonce`, `variation_nonce`, and new i18n strings to localized script data. Register new `product-variation.js` script. |
| `templates/products/product-form.php` | Add Attributes section, Variations section include, CSS classes `show_if_variable`, wrap pricing card for toggle |
| `assets/frontend/product.js` | Add product type toggle logic |

### Files to Create

| File | Purpose |
|---|---|
| `templates/products/product-attribute.php` | Single attribute row template |
| `templates/products/product-variations.php` | Variations wrapper template |
| `templates/products/product-variation.php` | Single variation row template |
| `assets/frontend/product-variation.js` | Attributes + Variations JavaScript management |

---

## 14. Data Flow Diagrams

### Creating a Variable Product (New)

```
User Flow:
──────────────────────────────────────────────────────────────────

1. Select "Variable" from Type dropdown
   └── JS hides pricing card, shows attributes + variations sections

2. Add Attributes
   ├── Select "Color" (taxonomy) or type custom name
   ├── Add values: Red, Blue, Green
   ├── Check "Used for variations" ✓
   └── Click "Save Attributes"
       └── AJAX: storesuite_save_attributes
           └── PHP: Creates WC_Product_Attribute objects → $product->set_attributes() → save()

3. Click "Generate Variations"
   └── AJAX: storesuite_generate_variations
       └── PHP: Reads variation attributes → generates combinations
           → Creates WC_Product_Variation for each → save()
           └── Returns success → JS calls loadVariations()

4. Edit Each Variation
   ├── Set prices, SKU, stock, image
   └── Click "Save Variations"
       └── AJAX: storesuite_save_variations
           └── PHP: Loops through $_POST arrays → updates each WC_Product_Variation → save()
               └── Calls WC_Product_Variable::sync() to update parent
```

### POST Data Structure for Variations

```
$_POST = [
    'product_id'              => 123,

    // Array indexed by loop position
    'variable_post_id'        => [0 => 456, 1 => 457, 2 => 458],
    'variable_enabled'        => [0 => 1, 1 => 1],           // Only checked ones
    'variable_sku'            => [0 => 'SKU-R', 1 => 'SKU-B', 2 => 'SKU-G'],
    'variable_regular_price'  => [0 => '29.99', 1 => '29.99', 2 => '34.99'],
    'variable_sale_price'     => [0 => '', 1 => '24.99', 2 => ''],
    'variable_stock_status'   => [0 => 'instock', 1 => 'instock', 2 => 'outofstock'],
    'variable_manage_stock'   => [0 => 'yes'],                // Only checked ones
    'variable_stock'          => [0 => '50', 1 => '', 2 => ''],
    'variable_weight'         => [0 => '', 1 => '', 2 => ''],
    'variable_description'    => [0 => '', 1 => 'Blue variant', 2 => ''],
    'upload_image_id'         => [0 => 789, 1 => 0, 2 => 0],

    // Variation attributes (dynamic keys based on attribute names)
    'attribute_pa_color'      => [0 => 'red', 1 => 'blue', 2 => 'green'],
    'attribute_pa_size'       => [0 => 'large', 1 => 'small', 2 => 'medium'],
];
```

---

## 15. Database Reference

After saving a variable product with variations, the data is stored as:

### `wp_posts`

| ID | post_type | post_parent | post_status |
|---|---|---|---|
| 123 | `product` | 0 | publish |
| 456 | `product_variation` | 123 | publish |
| 457 | `product_variation` | 123 | publish |

### `wp_postmeta` for parent (ID=123)

| meta_key | meta_value |
|---|---|
| `_product_attributes` | `a:1:{s:8:"pa_color";a:6:{...}}` (serialized) |
| `_price` | `24.99` (min of children) |
| `_price` | `29.99` (max of children) |

### `wp_postmeta` for variation (ID=456)

| meta_key | meta_value |
|---|---|
| `attribute_pa_color` | `red` |
| `_regular_price` | `29.99` |
| `_price` | `29.99` |
| `_stock_status` | `instock` |

---

## 16. Security Checklist

- [ ] All AJAX handlers verify nonce via `check_ajax_referer()`
- [ ] All AJAX handlers check `current_user_can( 'manage_woocommerce' )`
- [ ] Attribute names sanitized with `sanitize_title()` or `wc_clean()`
- [ ] Prices formatted with `wc_format_decimal()`
- [ ] Stock amounts validated with `wc_stock_amount()`
- [ ] All output escaped: `esc_attr()`, `esc_html()`, `esc_url()`
- [ ] No direct SQL — use WC CRUD methods (`$product->set_*()`, `$product->save()`)
- [ ] Variation ownership validated (variation's `post_parent` matches the product being edited)
- [ ] User cannot edit variations for products they don't own

---

## 17. Testing Checklist

### Add New Variable Product
- [ ] Select "Variable" type → pricing section hides
- [ ] Add taxonomy attribute (e.g., Color) → select terms → check "Used for variations"
- [ ] Add custom attribute (e.g., Material) → type pipe-separated values → check "Used for variations"
- [ ] Click "Save Attributes" → attributes persist
- [ ] Click "Generate Variations" → all combinations created
- [ ] Edit variation: set price, SKU, stock, image, description
- [ ] Click "Save Variations" → data persists
- [ ] View product on frontend → variation dropdowns work correctly

### Edit Existing Variable Product
- [ ] Load edit page → attributes section pre-populated
- [ ] Variations load via AJAX with correct data
- [ ] Modify variation price → save → confirm change persisted
- [ ] Add new attribute → regenerate variations → no duplicates created
- [ ] Remove a variation → confirm deleted

### Edge Cases
- [ ] Switch from "Variable" to "Simple" → variations preserved but hidden
- [ ] Switch from "Simple" to "Variable" → attributes/variations sections appear
- [ ] Product with 50+ variations → pagination works
- [ ] Custom attribute with special characters (quotes, ampersands)
- [ ] Empty attribute values handled gracefully
- [ ] Variation with "Any" attribute value (blank = matches all)

---

> **Note:** This document is a development blueprint. The code samples follow StoreSuite's existing patterns (AJAX via `admin-ajax.php`, `FormData`, SweetAlert2 for notifications, `storesuite_get_template_part()` for templates, Bootstrap grid for layout). Adapt class names, hook names, and file paths as needed during implementation.
