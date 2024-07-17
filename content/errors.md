# Donut Errors

## `:donut.error/schema-validation-error` {#:donut.error_schema-validation-error}

`donut.error/validate!` uses malli to for validation. See [the malli docs](https://github.com/metosin/malli) for more information on interpreting malli schema errors.

## `:donut.system/invalid-system` {#:donut.system_invalid-system}

See [your first system]({{< ref "/docs/system/tutorial/01-your-first-system" >}}) for a description of how to structure your system mpa.

## `:donut.system.validation/invalid-component-config` {#:donut.system.validation_invalid-component-config}

Your system is using [the validation plugin]({{< ref "/docs/system/#the-validation-plugin" >}}) to validate component configurations with a malli schema, and a component configuration was invalid.

See [the malli docs](https://github.com/metosin/malli) for more information on interpreting malli schema errors.

## `:donut.system.validation/invalid-instance` {#:donut.system.validation_invalid-instance}

Your system is using [the validation plugin]({{< ref "/docs/system/#the-validation-plugin" >}}) to validate component instances with a malli schema, and a component instance was invalid.

See [the malli docs](https://github.com/metosin/malli) for more information on interpreting malli schema errors.

