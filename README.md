# Bricks Form to Google Sheets Integration

This guide explains how to send Bricks form submissions directly to Google Sheets using a custom action in your WordPress child theme. Optionally, you can use a popup script to pre-fill product data.

---

## 1. Add Google Sheets Action to Bricks Form

Add the following code to your child theme's `functions.php` to enable the Google Sheets action in Bricks forms:

```php
// Add controls for Google Sheets action
add_filter('bricks/elements/form/controls', function($controls) {
    $controls['actions']['options']['googlesheets'] = esc_html__('Send to Google Sheets', 'bricks-child');
    $controls['googleSheetsEndpoint'] = [
        'group'       => 'googleSheets',
        'label'       => esc_html__('Endpoint URL', 'bricks-child'),
        'type'        => 'text',
        'description' => sprintf(
            '%s <a href="%s" target="_blank" rel="noopener">%s</a>',
            esc_html__('Paste your Google Sheets endpoint URL here.', 'bricks-child'),
            'https://github.com/petermann/bricks-google-sheets-integration/?tab=readme-ov-file#3-google-apps-script-for-receiving-webhook-data',
            esc_html__('Learn more', 'bricks-child')
        ),
    ];
    $controls['googleSheetsJson'] = [
        'group'       => 'googleSheets',
        'label'       => esc_html__('JSON Model', 'bricks-child'),
        'type'        => 'textarea',
        'description' => sprintf(
            '%s %s <a href="%s" target="_blank" rel="noopener">%s</a>',
            esc_html__('Use the field ID from your Bricks form for each value. Example: { "token": "YOUR_TOKEN_HERE", "name": "{{form_field_id}}", "email": "{{form_field_id}}", "phone": "{{form_field_id}}", "product": "{{form_field_id}}" }', 'bricks-child'),
            esc_html__('Set the correct field ID for each value to send the submitted data to Google Sheets. Include your security token as a literal value.', 'bricks-child'),
            'https://github.com/petermann/bricks-google-sheets-integration/?tab=readme-ov-file#2-example-json-model',
            esc_html__('Learn more', 'bricks-child')
        ),
    ];
    return $controls;
});

// Add control group for Google Sheets
add_filter('bricks/elements/form/control_groups', function($control_groups) {
    $control_groups['googleSheets'] = [
        'title'    => esc_html__('Google Sheets', 'bricks-child'),
        'required' => ['actions', '=', 'googlesheets'],
    ];
    return $control_groups;
});

// Custom action logic
add_action('bricks/form/action/googlesheets', function($form) {
    $settings = $form->get_settings();
    $fields   = (array) $form->get_fields();

    $endpoint = !empty($settings['googleSheetsEndpoint']) ? esc_url_raw($settings['googleSheetsEndpoint']) : '';
    $modelRaw = isset($settings['googleSheetsJson']) ? (string) $settings['googleSheetsJson'] : '';

    // Validate required fields
    if (empty($endpoint)) {
        error_log('Google Sheets: Endpoint URL is missing.');
        $form->set_result([
            'action'  => 'googlesheets',
            'type'    => 'error',
            'message' => current_user_can('manage_options')
                ? esc_html__('Google Sheets: Endpoint URL is required.', 'bricks-child')
                : esc_html__('An error occurred. Please try again later.', 'bricks-child')
        ]);
        return;
    }

    if (empty($modelRaw)) {
        error_log('Google Sheets: JSON Model is missing.');
        $form->set_result([
            'action'  => 'googlesheets',
            'type'    => 'error',
            'message' => current_user_can('manage_options')
                ? esc_html__('Google Sheets: JSON Model is required.', 'bricks-child')
                : esc_html__('An error occurred. Please try again later.', 'bricks-child')
        ]);
        return;
    }

    if (empty($fields)) {
        error_log('Google Sheets: No form fields submitted.');
        $form->set_result([
            'action'  => 'googlesheets',
            'type'    => 'error',
            'message' => current_user_can('manage_options')
                ? esc_html__('Google Sheets: No form fields submitted.', 'bricks-child')
                : esc_html__('An error occurred. Please try again later.', 'bricks-child')
        ]);
        return;
    }

    // Parse and validate JSON Model
    $model = json_decode($modelRaw, true);
    if (!is_array($model)) {
        error_log('Google Sheets: Invalid JSON Model - must be valid JSON.');
        $form->set_result([
            'action'  => 'googlesheets',
            'type'    => 'error',
            'message' => current_user_can('manage_options')
                ? esc_html__('Google Sheets: Invalid JSON Model (must be valid JSON).', 'bricks-child')
                : esc_html__('An error occurred. Please try again later.', 'bricks-child')
        ]);
        return;
    }

    // Map field_id => value
    $valueById = [];
    foreach ($fields as $key => $value) {
        if (strpos($key, 'form-field-') === 0) {
            $field_id = substr($key, 11);
            $valueById[$field_id] = is_array($value) ? implode(', ', $value) : (string) $value;
        }
    }

    // Replace placeholders only in string values
    $payload = [];
    foreach ($model as $k => $v) {
        if (is_string($v)) {
            $payload[$k] = preg_replace_callback('/{{([a-zA-Z0-9_]+)}}/', function($m) use ($valueById) {
                $id = $m[1];
                return isset($valueById[$id]) ? $valueById[$id] : '';
            }, $v);
        } else {
            $payload[$k] = $v;
        }
    }

    // Send request to Google Sheets
    $response = wp_remote_post($endpoint, [
        'timeout'     => 8,
        'redirection' => 0, // <- importante
        'headers'     => [
            'Content-Type' => 'application/json; charset=utf-8',
            'Accept'       => 'application/json',
        ],
        'body'        => wp_json_encode($payload),
        'data_format' => 'body',
    ]);

    // Handle request errors
    if (is_wp_error($response)) {
        error_log('Google Sheets error: ' . $response->get_error_message());
        $form->set_result([
            'action'  => 'googlesheets',
            'type'    => 'error',
            'message' => current_user_can('manage_options')
                ? esc_html__('Google Sheets error: ', 'bricks-child') . $response->get_error_message()
                : esc_html__('An error occurred. Please try again later.', 'bricks-child')
        ]);
        return;
    }

    // Handle HTTP status codes
    $code = wp_remote_retrieve_response_code($response);
    if (!in_array($code, [200, 302], true)) {
        error_log('Google Sheets HTTP ' . $code . ' body: ' . wp_remote_retrieve_body($response));
        $form->set_result([
            'action'  => 'googlesheets',
            'type'    => 'error',
            'message' => current_user_can('manage_options')
                ? esc_html__('Google Sheets returned HTTP ', 'bricks-child') . $code
                : esc_html__('An error occurred. Please try again later.', 'bricks-child')
        ]);
        return;
    }

    // Success - optionally log for debugging
    // error_log('Google Sheets: Successfully sent form data. HTTP ' . $code);
});
```

**Instructions:**
- Paste the code above into your child theme's `functions.php`.
- In the Bricks form builder, select the "Send to Google Sheets" action.
- Fill in the Endpoint URL and JSON Model fields as described below.

---

## 2. Example JSON Model

Use this template in the JSON Model field. Replace the field IDs with those from your form and set your security token:

```json
{
  "token": "YOUR_SUPER_SECRET_TOKEN_HERE",
  "name": "{{form_field_name}}",
  "email": "{{form_field_email}}",
  "phone": "{{form_field_phone}}",
  "product": "{{form_field_product}}",
  "message": "{{form_field_message}}"
}
```

**Important notes:**
- The JSON Model must be **valid JSON syntax**.
- Replace `YOUR_SUPER_SECRET_TOKEN_HERE` with your actual security token (same token used in Google Apps Script).
- Replace `form_field_name`, `form_field_email`, etc., with the actual field IDs from your Bricks form.
- You can find the field ID by inspecting the form element settings in Bricks Builder.
- Placeholders that don't match any form field will be replaced with empty strings.

---

## 3. Google Apps Script for Receiving Webhook Data

Create a new Google Apps Script attached to your Google Sheet and paste the following code:

```javascript
const SHEET_NAME = 'Leads'; // Change this to your preferred sheet name
const TOKEN = 'YOUR_SUPER_SECRET_TOKEN_HERE'; // Set the same token used in your JSON Model

function doPost(e) {
  try {
    // Get the specific sheet by name (more reliable than getActiveSheet)
    const sheet = SpreadsheetApp.getActiveSpreadsheet().getSheetByName(SHEET_NAME);
    if (!sheet) {
      throw new Error('Sheet not found: ' + SHEET_NAME);
    }

    // Parse the incoming data
    const data = JSON.parse((e.postData && e.postData.contents) ? e.postData.contents : '{}');

    // Token validation from request body
    if (TOKEN && String(data.token || '') !== TOKEN) {
      return ContentService
        .createTextOutput(JSON.stringify({ result: "error", message: "unauthorized" }))
        .setMimeType(ContentService.MimeType.JSON);
    }

    // Helper function to process arrays (for multi-select fields)
    function processValue(value) {
      if (Array.isArray(value)) {
        return value.join(', ');
      }
      return value || '';
    }

    // Create row data - customize these fields based on your JSON model
    const row = [
      processValue(data.name),
      processValue(data.email),
      processValue(data.phone),
      processValue(data.product),
      processValue(data.message),
      new Date() // Timestamp
    ];

    // Add the row to the sheet
    sheet.appendRow(row);

    // Return success response
    return ContentService
      .createTextOutput(JSON.stringify({ result: "success" }))
      .setMimeType(ContentService.MimeType.JSON);

  } catch (err) {
    // Log error for debugging
    console.error("Apps Script Error:", err);
    
    // Return error response
    return ContentService
      .createTextOutput(JSON.stringify({ result: "error", message: String(err) }))
      .setMimeType(ContentService.MimeType.JSON);
  }
}
```

**Setup Steps:**
1. Open your Google Sheet and create a sheet named "Leads" (or change `SHEET_NAME` in the script).
2. Add headers to your sheet: `Name`, `Email`, `Phone`, `Product`, `Message`, `Timestamp`.
3. Go to **Extensions > Apps Script**.
4. Delete any existing code and paste the script above.
5. Set the `TOKEN` constant to the same value used in your JSON Model.
6. Save the project with a meaningful name.
7. Deploy as a web app:
   - Click **Deploy > New deployment**
   - Choose **Web app** as the type
   - Set **Execute as**: Me (your account)
   - Set **Who has access**: Anyone (for the form to work)
   - Click **Deploy**
8. Copy the web app URL to use as your webhook endpoint in Bricks.

---

## 4. Security Configuration

The security token is now sent in the request body instead of the URL, providing better security:

**Generate a strong token:**
- Use a password generator to create a random 32+ character string
- Include letters, numbers, and special characters
- Keep this token secret and don't share it publicly

**Example token:**
```
Kj8mN2pQ5wR9tY6uI3oP7aS1dF4gH0zX
```

Make sure the same token is set in both your JSON Model and your Google Apps Script `TOKEN` constant.

---

## 5. (Optional) Open Bricks Form Popup and Set Product

Add this JavaScript to your site if you want to open a popup and pre-fill a product field:

```javascript
function openPopupSubscribe(product) {
  // Replace with your actual popup ID
  let popupId = 1; 
  
  const productName = product || 'No product selected.';
  
  // Open the popup first
  bricksOpenPopup(popupId);
  
  // Wait a moment for the popup to load, then set the product field
  setTimeout(function() {
    const productField = document.querySelector('#brxe-popup-' + popupId + ' input[name="product"]');
    if (productField) {
      productField.value = productName;
    } else {
      console.warn('Product input field not found in popup #' + popupId);
    }
  }, 100);
}
```

**Usage example:**
```html
<button onclick="openPopupSubscribe('Premium Plan')">Subscribe to Premium</button>
<button onclick="openPopupSubscribe('Basic Plan')">Subscribe to Basic</button>
```

---

## 6. Troubleshooting

### Common Issues:

**Form submission fails silently:**
- Check the browser console for JavaScript errors
- Verify the Google Apps Script URL is correct
- Ensure the JSON Model syntax is valid

**"Unauthorized" error:**
- Verify the token matches exactly in both JSON Model and Apps Script
- Check that the token is properly set without extra spaces

**Data not appearing in sheets:**
- Verify the sheet name matches the `SHEET_NAME` in the script
- Check the Apps Script logs: **Executions** tab in the script editor
- Ensure the web app is deployed with "Anyone" access

**Field values not replacing placeholders:**
- Verify field IDs in your JSON Model match the actual Bricks form field IDs
- Check that field IDs don't contain special characters or spaces

### Debugging Tips:

1. **Check WordPress error logs** for detailed error messages (only visible to administrators)
2. **Monitor Apps Script execution logs** in the Google Apps Script editor
3. **Test with a simple JSON model** first: `{"token": "YOUR_TOKEN", "test": "{{field_id}}"}`
4. **Verify form field IDs** by inspecting the HTML or Bricks element settings

---

## 7. Advanced Configuration

### Multiple Sheets for Different Forms:

Modify the Apps Script to write to different sheets based on a form identifier:

```javascript
const data = JSON.parse((e.postData && e.postData.contents) ? e.postData.contents : '{}');
const formType = data.form_type || 'default';

let sheetName;
switch(formType) {
  case 'contact':
    sheetName = 'Contact Form';
    break;
  case 'newsletter':
    sheetName = 'Newsletter';
    break;
  default:
    sheetName = 'Leads';
}

const sheet = SpreadsheetApp.getActiveSpreadsheet().getSheetByName(sheetName);
```

### Adding UTM Parameters and Metadata:

Extend your JSON model to include additional data:

```json
{
  "token": "YOUR_SUPER_SECRET_TOKEN_HERE",
  "name": "{{form_field_name}}",
  "email": "{{form_field_email}}",
  "form_type": "contact",
  "page_url": "{{current_url}}",
  "utm_source": "{{utm_source}}",
  "utm_campaign": "{{utm_campaign}}"
}
```

Note: You'll need to add custom code to capture UTM parameters and current URL in WordPress.

---

## Key Security Improvements

This version includes several security and reliability improvements:

- **Token in request body**: Security token is sent in the JSON payload instead of URL parameters
- **Proper JSON handling**: Uses `json_decode` and `wp_json_encode` for safe processing
- **HTTP timeout and status checking**: Prevents hanging requests and proper error handling  
- **Fixed sheet targeting**: Uses a specific sheet name instead of "active" sheet
- **Better error logging**: Detailed logs for administrators, generic messages for users
- **Robust placeholder replacement**: Safer handling of form field substitution

---

## References

- [Bricks Builder Documentation](https://academy.bricksbuilder.io/article/form-element/)
- [Google Apps Script Web Apps](https://developers.google.com/apps-script/guides/web)
- [WordPress HTTP API](https://developer.wordpress.org/reference/functions/wp_remote_post/)

---

By [Ivan Petermann](https://ivanpetermann.com/)