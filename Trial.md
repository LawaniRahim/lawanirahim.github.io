# DevArca Visual Color Editor — Complete Setup & Deployment Guide

This guide walks you through setting up and deploying the **DevArca Visual Color Editor** on your WordPress site running the **Astra Child Theme**, **Easy Digital Downloads (EDD)**, **Elementor**, and **Cloudflare R2**.

---

## 📋 Table of Contents
1. [Prerequisites & System Requirements](#1-prerequisites--system-requirements)
2. [Directory Structure Overview](#2-directory-structure-overview)
3. [Installing the Astra Child Theme Files](#3-installing-the-astra-child-theme-files)
4. [Setting Up Cloudflare R2 Storage](#4-setting-up-cloudflare-r2-storage)
5. [Configuring `wp-config.php` Environment Variables](#5-configuring-wp-configphp-environment-variables)
6. [Easy Digital Downloads (EDD) Product Configuration](#6-easy-digital-downloads-edd-product-configuration)
7. [Product Package Structure & Uploading Assets](#7-product-package-structure--uploading-assets)
8. [Embedding the Editor in WordPress & Elementor](#8-embedding-the-editor-in-wordpress--elementor)
9. [Guest Purchase & Entitlement Verification Flow](#9-guest-purchase--entitlement-verification-flow)
10. [Troubleshooting & Verification Checklist](#10-troubleshooting--verification-checklist)

---

## 1. Prerequisites & System Requirements

Ensure your hosting environment meets the following specifications:
- **WordPress**: Version 6.0 or higher
- **PHP**: Version 7.4 or higher (8.1+ recommended) with `cURL` and `ZipArchive` extensions enabled
- **Theme**: Active **Astra Child Theme** (parent Astra theme installed)
- **Plugins**:
  - **Easy Digital Downloads (EDD)** (free or Pro version)
  - **Elementor** (free or Pro version, optional but recommended for visual page building)
- **Cloudflare Account**: With **R2 Object Storage** enabled (includes a generous free tier of 10GB storage and millions of monthly operations)

---

## 2. Directory Structure Overview

Ensure your `wp-content/themes/astra-child` folder contains the following files:

```text
wp-content/themes/astra-child/
├── style.css                           # Child theme declaration
├── functions.php                       # Theme entry point that boots the editor
├── devarca-color-editor-api.php        # Backend REST API, SigV4 R2 Client, ZIP compiler, Entitlement Gate
├── devarca-color-editor-frontend.php   # Shortcode [devarca_color_editor] & conditional asset enqueuer
└── assets/
    ├── css/
    │   └── color-editor.css            # Scoped dark aesthetic, responsive stage, custom color inputs
    └── js/
        └── color-editor.js             # Client state manager, live iframe sandbox, ZIP exporter
```

---

## 3. Installing the Astra Child Theme Files

### Step 3.1: Copy Theme Files
Copy the editor files into your active child theme directory:
1. Upload `devarca-color-editor-api.php` to `/wp-content/themes/astra-child/`.
2. Upload `devarca-color-editor-frontend.php` to `/wp-content/themes/astra-child/`.
3. Upload `assets/css/color-editor.css` to `/wp-content/themes/astra-child/assets/css/`.
4. Upload `assets/js/color-editor.js` to `/wp-content/themes/astra-child/assets/js/`.

### Step 3.2: Verify `functions.php`
Open `/wp-content/themes/astra-child/functions.php` and ensure it includes both modules:

```php
<?php
/**
 * Astra Child Theme Functions
 *
 * @package AstraChild
 */

if ( ! defined( 'ABSPATH' ) ) {
    exit;
}

// Load DevArca Color Editor Backend API & Entitlement Gate
require_once get_stylesheet_directory() . '/devarca-color-editor-api.php';

// Load DevArca Color Editor Frontend Handler & Elementor Integration
require_once get_stylesheet_directory() . '/devarca-color-editor-frontend.php';
```

When activated, this will automatically:
- Register the REST API endpoints:
  - `POST /wp-json/devarca/v1/editor-data`
  - `POST /wp-json/devarca/v1/generate-css`
- Register the shortcode: `[devarca_color_editor]`

---

## 4. Setting Up Cloudflare R2 Storage

Cloudflare R2 provides private, zero-egress-fee storage for your master component source files (`style.css`, `schema.json`, `preview.html`, `meta.json`, and optional `script.js`).

### Step 4.1: Create an R2 Bucket
1. Log in to your **Cloudflare Dashboard**.
2. In the left navigation, click **R2**.
3. Click **Create bucket**.
4. Name your bucket (e.g., `devarca-products`).
5. Choose **Default location** or select your preferred region.
6. Leave the bucket set to **Private** (do not enable public access; our server-side SigV4 client signs requests securely).

### Step 4.2: Generate API Tokens with R2 Permissions
1. On the R2 overview page in Cloudflare, click **Manage R2 API Tokens** in the right sidebar.
2. Click **Create API Token**.
3. Choose the permission: **Object Read & Write** (or **Object Read Only** for WordPress production).
4. Specify your bucket name (`devarca-products`).
5. Click **Create API Token**.
6. Cloudflare will display:
   - **Access Key ID**
   - **Secret Access Key**
   - **Endpoint** (format: `https://<ACCOUNT_ID>.r2.cloudflarestorage.com`)
7. **Save these credentials securely** — the secret key will only be shown once.

---

## 5. Configuring `wp-config.php` Environment Variables

Open your root `wp-config.php` file on the WordPress server (located in your web root) and add the R2 configuration constants **above** the `/* That's all, stop editing! Happy publishing. */` line:

```php
/**
 * DevArca Visual Color Editor - Cloudflare R2 Private Storage Config
 */
define( 'DEVARCA_R2_ACCOUNT_ID',        'your_cloudflare_account_id_here' );
define( 'DEVARCA_R2_BUCKET',            'devarca-products' );
define( 'DEVARCA_R2_ACCESS_KEY_ID',     'your_r2_access_key_id_here' );
define( 'DEVARCA_R2_SECRET_ACCESS_KEY', 'your_r2_secret_access_key_here' );
define( 'DEVARCA_R2_ENDPOINT',          'https://your_cloudflare_account_id_here.r2.cloudflarestorage.com' );
```

> **Local Development Fallback**: If `DEVARCA_R2_BUCKET` is omitted or empty, the API automatically falls back to checking the local `/products/{product-slug}/` directory inside your WordPress installation.

---

## 6. Easy Digital Downloads (EDD) Product Configuration

The editor protects your paid products by checking customer order history.

### Step 6.1: Create or Edit a Download in EDD
1. In your WordPress admin, navigate to **Downloads** > **Add New** (or edit an existing download).
2. Set the **Title** (e.g., `Hero Section 1`).
3. Set the **Slug** in the permalink area (e.g., `hero-section-1`). 
   *Note: This slug must match the folder name in your R2 bucket.*
4. Set the **Price** (e.g., `$19.00`). If price is `$0.00`, the editor treats it as a free product and unlocks it immediately for all visitors.
5. In **Download Files**, upload the default pre-compiled package or placeholder zip.
6. Publish the Download.

---

## 7. Product Package Structure & Uploading Assets

Each customizable product requires four files, with an optional `script.js` for interactions, stored under `products/{product_slug}/`:

```text
products/
└── hero-section-1/
    ├── meta.json       # Product name, version, and licensing information
    ├── schema.json     # Variable definitions (solid colors, gradients, labels, defaults)
    ├── preview.html    # Raw HTML markup displayed inside the preview iframe
    ├── style.css       # Clean CSS stylesheet with CSS variables declared on :root
    └── script.js       # Optional component interactions, isolated in the preview iframe
```

### 7.1 Example `schema.json`
Defines which CSS variables are editable in the UI:

```json
{
  "name": "Hero Section 1",
  "version": "1.0.0",
  "groups": [
    {
      "id": "background",
      "label": "Background",
      "fields": [
        {
          "var": "--hero-bg",
          "label": "Background color/gradient",
          "type": "gradient",
          "default": {
            "type": "linear",
            "angle": 135,
            "stops": [
              { "color": "#0F172A", "position": 0 },
              { "color": "#1E293B", "position": 100 }
            ]
          }
        }
      ]
    },
    {
      "id": "typography",
      "label": "Typography",
      "fields": [
        {
          "var": "--hero-heading-color",
          "label": "Heading color",
          "type": "solid",
          "default": "#F8FAFC"
        },
        {
          "var": "--hero-subtext-color",
          "label": "Subtext color",
          "type": "solid",
          "default": "#94A3B8"
        }
      ]
    },
    {
      "id": "cta",
      "label": "Primary Button",
      "fields": [
        {
          "var": "--hero-cta-bg",
          "label": "Button background",
          "type": "solid",
          "default": "#3B82F6"
        },
        {
          "var": "--hero-cta-text",
          "label": "Button text color",
          "type": "solid",
          "default": "#FFFFFF"
        }
      ]
    }
  ]
}
```

### 7.2 Uploading to Cloudflare R2
In Cloudflare R2 Dashboard > `devarca-products`:
1. Click **Upload** > **Upload Folder** (or upload files individually).
2. Ensure the object path in R2 matches:
   - `products/hero-section-1/meta.json`
   - `products/hero-section-1/schema.json`
   - `products/hero-section-1/preview.html`
   - `products/hero-section-1/style.css`
   - `products/hero-section-1/script.js` (optional)

---

## 8. Embedding the Editor in WordPress & Elementor

You can embed the visual editor into any page, template, or custom post type.

### Method A: Using the WordPress Block Editor (Gutenberg)
1. Edit your page.
2. Add a **Shortcode block**.
3. Insert:
   ```text
   [devarca_color_editor product="hero-section-1"]
   ```
4. Save and publish.

### Method B: Using Elementor
1. Open your target page in **Elementor**.
2. Search for the **Shortcode** widget in the left panel.
3. Drag the Shortcode widget onto your layout (we recommend setting the section/container to **Full Width** or boxed with max-width `1280px`).
4. In the Shortcode input box, enter:
   ```text
   [devarca_color_editor product="hero-section-1"]
   ```
5. Click **Update** to publish.

> **Dynamic Download ID Support**: You can also reference a product by its EDD Download ID:
> ```text
> [devarca_color_editor download_id="4821"]
> ```

---

## 9. Guest Purchase & Entitlement Verification Flow

The editor automatically supports two types of authorized customers:

### Scenario 1: Logged-in Customer
- If the user is logged into their WordPress customer account and previously bought the product in EDD, the editor unlocks automatically with zero friction.

### Scenario 2: Guest Checkout (Direct Link or Purchase Receipt)
- EDD allows guests to checkout using their email address.
- You can provide customers with a direct unlock URL in their EDD purchase confirmation email:
  ```text
  https://yoursite.com/editor/?product=hero-section-1&email=customer@example.com&payment_key=abc123paymentkey
  ```
- If a guest visits without parameters, the editor displays a clean lock card with:
  1. A button to **Purchase License on EDD**.
  2. A **Guest Verification toggle** allowing them to enter their billing email and payment key to unlock the editor on the spot.
- Verification uses constant-time hash comparison (`hash_equals`) to protect against timing attacks.

---

## 10. Troubleshooting & Verification Checklist

| Symptom | Cause | Solution |
| :--- | :--- | :--- |
| **"Initializing editor..." hangs** | REST API route blocked or nonce missing | Ensure your permalinks are enabled under *Settings > Permalinks* (set to `Post name`). Verify the REST endpoint `/wp-json/devarca/v1/editor-data` is accessible. |
| **"Premium Template Locked" displays for buyer** | User not logged in, or payment status not completed | Check EDD Payments to ensure the order is marked **Completed**. For guests, verify the email and payment key match the order. |
| **"File style.css not found"** | Missing files in R2 bucket | Verify the exact key in Cloudflare R2: `products/{product-slug}/style.css`. Object keys are case-sensitive. |
| **Rate limit toast notification** | More than 20 exports per hour | The built-in rate limiter protects server CPU. Wait for the 1-hour window or increase `RATE_LIMIT_MAX` in `devarca-color-editor-api.php`. |
| **Downloaded ZIP contains default styles** | CSS compiler regex mismatch | Ensure your `style.css` declares variables in the format `--var-name: <value>;` inside `:root { ... }`. |
| **Preview sizing on mobile/tablet** | Browser viewport cached | The editor features dynamic `fitIframeHeight` calculation. Hard refresh your browser (`Ctrl+Shift+R` or `Cmd+Shift+R`) to load the latest JS. |

---

### 🎉 You're All Set!
Your customers can now customize theme colors live in the responsive previewer and export clean, production-ready CSS packages in 1-click.
