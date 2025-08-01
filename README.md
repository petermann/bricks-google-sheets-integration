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
            esc_html__('Use the field ID from your Bricks form for each value. Example: { "name": "{{form_field_id}}", "email": "{{form_field_id}}", "phone": "{{form_field_id}}", "product": "{{form_field_id}}" }', 'bricks-child'),
            esc_html__('Set the correct field ID for each value to send the submitted data to Google Sheets.', 'bricks-child'),
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
    $fields   = $form->get_fields();
    if (empty($settings['googleSheetsEndpoint'])) {
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
    if (empty($settings['googleSheetsJson'])) {
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
    // Replace {{field_id}} with form values
    $json_model = $settings['googleSheetsJson'];
    if (
        !is_string($json_model) ||
        trim($json_model)[0] !== '{' ||
        trim($json_model)[strlen(trim($json_model))-1] !== '}' ||
        !preg_match('/{{[a-zA-Z0-9_]+}}/', $json_model)
    ) {
        error_log('Google Sheets: Invalid JSON model!');
        $form->set_result([
            'action'  => 'googlesheets',
            'type'    => 'error',
            'message' => current_user_can('manage_options')
                ? esc_html__('Google Sheets: Invalid JSON model.', 'bricks-child')
                : esc_html__('An error occurred. Please try again later.', 'bricks-child')
        ]);
        return;
    }
    foreach ($fields as $key => $value) {
        if (strpos($key, 'form-field-') === 0) {
            $field_id = substr($key, 11);
            $replace_value = is_array($value) ? implode(', ', $value) : $value;
            $replace_value = sanitize_text_field($replace_value);
            $json_model = str_replace('{{' . $field_id . '}}', $replace_value, $json_model);
        }
    }
    $json_model = preg_replace('/{{[a-zA-Z0-9_]+}}/', '""', $json_model);
    $endpoint = esc_url_raw($settings['googleSheetsEndpoint']);
    $response = wp_remote_post($endpoint, [
        'method'      => 'POST',
        'headers'     => ['Content-Type' => 'application/json'],
        'body'        => $json_model,
        'data_format' => 'body'
    ]);
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
});
```

**Instructions:**
- Paste the code above into your child theme’s `functions.php`.
- In the Bricks form builder, select the “Send to Google Sheets” action.
- Fill in the Endpoint URL and JSON Model fields as described above.

---

## 2. Example JSON Model

Use this template in the JSON Model field. Replace the field IDs with those from your form:

```
{
  "name": "{{form_field_id}}",
  "email": "{{form_field_id}}",
  "phone": "{{form_field_id}}",
  "product": "{{form_field_id}}"
}
```

---

## 3. Google Apps Script for Receiving Webhook Data

Create a new Google Apps Script attached to your Google Sheet and paste the following code:

```javascript
function doPost(e) {
  var sheet = SpreadsheetApp.getActiveSheet();
  var data = JSON.parse(e.postData.contents);
  function processArray(value) {
    if (Array.isArray(value)) {
      return value.join(', ');
    }
    return value;
  }
  var row = [
    processArray(data.name),
    processArray(data.email),
    processArray(data.phone),
    processArray(data.product),
    new Date()
  ];
  sheet.appendRow(row);
  return ContentService.createTextOutput(JSON.stringify({ result: "success" }))
    .setMimeType(ContentService.MimeType.JSON);
}
```

**Setup Steps:**
1. Open your Google Sheet.
2. Go to Extensions > Apps Script.
3. Paste the code above and save.
4. Deploy as a web app (set access to "Anyone, even anonymous").
5. Copy the web app URL to use as your webhook endpoint in Bricks.


---

## 4. (Optional) Open Bricks Form Popup and Set Product

Add this JavaScript to your site if you want to open a popup and pre-fill a product field:

```js
function openPopupSubscribe(product) {
  let popupId = 1; // Replace with your actual popup ID
  const productName = product || 'No product selected.';
  const productField = document.querySelector('input[name="product"]');
  if (productField) {
    productField.value = productName;
  } else {
    console.warn('Hidden input with name "product" was not found in the DOM.');
  }
  bricksOpenPopup(popupId);
}
```

---

## References

- [Bricks Builder Documentation](https://academy.bricksbuilder.io/article/form-element/)
- [Google Apps Script Web Apps](https://developers.google.com/apps-script/guides/web)

---

By [Ivan Petermann](https://ivanpetermann.com/)
