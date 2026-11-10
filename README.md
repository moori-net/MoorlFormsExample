# Developer example to create own add-ons for "Form Builder 2"

This repository demonstrates how to register a product-related form type, a newsletter form action and a custom field type for Form Builder 2. It also contains administration overrides, product buy-widget templates and a small storefront JavaScript plugin.

## Compatibility status

The manifest declares Shopware `6.7.*`, Foundation `1.7.*` and MoorlForms `2.7.*`, but the example action and element still use older method signatures. Loading the actual classes with the current local MoorlForms base classes fails with PHP declaration compatibility errors. Migrate these contracts before activating or copying the example:

- `FormActionExample::init()` and `process()` need the current request/context parameters; the base class no longer stores `$request` or `$salesChannelContext` as properties.
- `ElementTypeExample::init()` and `validate()` need the current form, request, context and validator parameters.
- The custom element template references MoorlFormsClassic, which is not listed as a Composer requirement here.

The intended action registers a newsletter subscription, and the intended element validation calls the external VIES SOAP service. They are functional integration examples with side effects, not inert demonstration handlers. The product UI and administration overrides have not been verified in a browser for this source state.

See [CODEX.md](CODEX.md) for the complete source map, registration names, data flow, configuration fields and verified limitations.
