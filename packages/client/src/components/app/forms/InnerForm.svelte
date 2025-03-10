<script>
  import { setContext, getContext } from "svelte"
  import { derived, get, writable } from "svelte/store"
  import { createValidatorFromConstraints } from "./validation"
  import { Helpers } from "@budibase/bbui"

  export let dataSource
  export let disabled = false
  export let initialValues
  export let size
  export let schema
  export let table
  export let disableValidation = false

  const component = getContext("component")
  const { styleable, Provider, ActionTypes } = getContext("sdk")

  let fields = []
  const currentStep = writable(1)
  const formState = writable({
    values: {},
    errors: {},
    valid: true,
    currentStep: 1,
  })

  // Reactive derived stores to derive form state from field array
  $: values = deriveFieldProperty(fields, f => f.fieldState.value)
  $: errors = deriveFieldProperty(fields, f => f.fieldState.error)
  $: enrichments = deriveBindingEnrichments(fields)
  $: valid = !Object.values($errors).some(error => error != null)

  // Derive whether the current form step is valid
  $: currentStepValid = derived(
    [currentStep, ...fields],
    ([currentStepValue, ...fieldsValue]) => {
      return !fieldsValue
        .filter(f => f.step === currentStepValue)
        .some(f => f.fieldState.error != null)
    }
  )

  // Update form state store from derived stores
  $: {
    formState.set({
      values: $values,
      errors: $errors,
      valid,
      currentStep: $currentStep,
    })
  }

  // Derive value of whole form
  $: formValue = deriveFormValue(initialValues, $values, $enrichments)

  // Create data context to provide
  $: dataContext = {
    ...formValue,

    // These static values are prefixed to avoid clashes with actual columns
    __value: formValue,
    __valid: valid,
    __currentStep: $currentStep,
    __currentStepValid: $currentStepValid,
  }

  // Generates a derived store from an array of fields, comprised of a map of
  // extracted values from the field array
  const deriveFieldProperty = (fieldStores, getProp) => {
    return derived(fieldStores, fieldValues => {
      const reducer = (map, field) => ({ ...map, [field.name]: getProp(field) })
      return fieldValues.reduce(reducer, {})
    })
  }

  // Derives any enrichments which need to be made so that bindings work for
  // special data types like attachments. Relationships are currently not
  // handled as we don't have the primaryDisplay field that is required.
  const deriveBindingEnrichments = fieldStores => {
    return derived(fieldStores, fieldValues => {
      let enrichments = {}
      fieldValues.forEach(field => {
        if (field.type === "attachment") {
          const value = field.fieldState.value
          let url = null
          if (Array.isArray(value) && value[0] != null) {
            url = value[0].url
          }
          enrichments[`${field.name}_first`] = url
        }
      })
      return enrichments
    })
  }

  // Derive the overall form value and deeply set all field paths so that we
  // can support things like JSON fields.
  const deriveFormValue = (initialValues, values, enrichments) => {
    let formValue = Helpers.cloneDeep(initialValues || {})

    // We need to sort the keys to avoid a JSON field overwriting a nested field
    const sortedFields = Object.entries(values || {})
      .map(([key, value]) => {
        const field = getField(key)
        return {
          key,
          value,
          lastUpdate: get(field).fieldState?.lastUpdate || 0,
        }
      })
      .sort((a, b) => {
        return a.lastUpdate > b.lastUpdate
      })

    // Merge all values and enrichments into a single value
    sortedFields.forEach(({ key, value }) => {
      Helpers.deepSet(formValue, key, value)
    })
    Object.entries(enrichments || {}).forEach(([key, value]) => {
      Helpers.deepSet(formValue, key, value)
    })
    return formValue
  }

  // Searches the field array for a certain field
  const getField = name => {
    return fields.find(field => get(field).name === name)
  }

  const formApi = {
    registerField: (
      field,
      type,
      defaultValue = null,
      fieldDisabled = false,
      validationRules,
      step = 1
    ) => {
      if (!field) {
        return
      }

      // Create validation function based on field schema
      const schemaConstraints = schema?.[field]?.constraints
      const validator = disableValidation
        ? null
        : createValidatorFromConstraints(
            schemaConstraints,
            validationRules,
            field,
            table
          )

      // If we've already registered this field then keep some existing state
      let initialValue = Helpers.deepGet(initialValues, field) ?? defaultValue
      let initialError = null
      let fieldId = `id-${Helpers.uuid()}`
      const existingField = getField(field)
      if (existingField) {
        const { fieldState } = get(existingField)
        fieldId = fieldState.fieldId

        // Determine the initial value for this field, reusing the current
        // value if one exists
        initialValue = fieldState.value ?? initialValue

        // If this field has already been registered and we previously had an
        // error set, then re-run the validator to see if we can unset it
        if (fieldState.error) {
          initialError = validator?.(initialValue)
        }
      }

      // Auto columns are always disabled
      const isAutoColumn = !!schema?.[field]?.autocolumn

      // Construct field info
      const fieldInfo = writable({
        name: field,
        type,
        step: step || 1,
        fieldState: {
          fieldId,
          value: initialValue,
          error: initialError,
          disabled: disabled || fieldDisabled || isAutoColumn,
          defaultValue,
          validator,
          lastUpdate: Date.now(),
        },
        fieldApi: makeFieldApi(field, defaultValue),
        fieldSchema: schema?.[field] ?? {},
      })

      // Add this field
      if (existingField) {
        const otherFields = fields.filter(info => get(info).name !== field)
        fields = [...otherFields, fieldInfo]
      } else {
        fields = [...fields, fieldInfo]
      }

      return fieldInfo
    },
    validate: (onlyCurrentStep = false) => {
      let valid = true
      let validationFields = fields

      // Reduce fields to only the current step if required
      if (onlyCurrentStep) {
        validationFields = fields.filter(f => get(f).step === get(currentStep))
      }

      // Validate fields and check if any are invalid
      validationFields.forEach(field => {
        if (!get(field).fieldApi.validate()) {
          valid = false
        }
      })
      return valid
    },
    clear: () => {
      // Clear the form by clearing each individual field
      fields.forEach(field => {
        get(field).fieldApi.clearValue()
      })
    },
    changeStep: ({ type, number }) => {
      if (type === "next") {
        currentStep.update(step => step + 1)
      } else if (type === "prev") {
        currentStep.update(step => Math.max(1, step - 1))
      } else if (type === "first") {
        currentStep.set(1)
      } else if (type === "specific" && number && !isNaN(number)) {
        currentStep.set(number)
      }
    },
    setStep: step => {
      if (step) {
        currentStep.set(step)
      }
    },
  }

  // Creates an API for a specific field
  const makeFieldApi = field => {
    // Sets the value for a certain field and invokes validation
    const setValue = (value, skipCheck = false) => {
      const fieldInfo = getField(field)
      const { fieldState } = get(fieldInfo)
      const { validator } = fieldState

      // Skip if the value is the same
      if (!skipCheck && fieldState.value === value) {
        return
      }

      // Update field state
      const error = validator?.(value)
      fieldInfo.update(state => {
        state.fieldState.value = value
        state.fieldState.error = error
        state.fieldState.lastUpdate = Date.now()
        return state
      })

      return !error
    }

    // Clears the value of a certain field back to the initial value
    const clearValue = () => {
      const fieldInfo = getField(field)
      const { fieldState } = get(fieldInfo)
      const newValue = initialValues[field] ?? fieldState.defaultValue

      // Update field state
      fieldInfo.update(state => {
        state.fieldState.value = newValue
        state.fieldState.error = null
        state.fieldState.lastUpdate = Date.now()
        return state
      })
    }

    // Updates the validator rules for a certain field
    const updateValidation = validationRules => {
      const fieldInfo = getField(field)
      const { fieldState } = get(fieldInfo)
      const { value, error } = fieldState

      // Create new validator
      const schemaConstraints = schema?.[field]?.constraints
      const validator = disableValidation
        ? null
        : createValidatorFromConstraints(
            schemaConstraints,
            validationRules,
            field,
            table
          )

      // Update validator
      fieldInfo.update(state => {
        state.fieldState.validator = validator
        return state
      })

      // If there is currently an error, run the validator again in case
      // the error should be cleared by the new validation rules
      if (error) {
        setValue(value, true)
      }
    }

    // Updates the disabled state of a certain field
    const setDisabled = fieldDisabled => {
      const fieldInfo = getField(field)

      // Auto columns are always disabled
      const isAutoColumn = !!schema?.[field]?.autocolumn

      // Update disabled state
      fieldInfo.update(state => {
        state.fieldState.disabled = disabled || fieldDisabled || isAutoColumn
        return state
      })
    }

    return {
      setValue,
      clearValue,
      updateValidation,
      setDisabled,
      validate: () => {
        // Validate the field by force setting the same value again
        const { fieldState } = get(getField(field))
        return setValue(fieldState.value, true)
      },
    }
  }

  // Provide form state and api for full control by children
  setContext("form", {
    formState,
    formApi,

    // Data source is needed by attachment fields to be able to upload files
    // to the correct table ID
    dataSource,
  })

  // Provide form step context so that forms without any step components
  // register their fields to step 1
  setContext("form-step", writable(1))

  // Action context to pass to children
  const actions = [
    { type: ActionTypes.ValidateForm, callback: formApi.validate },
    { type: ActionTypes.ClearForm, callback: formApi.clear },
    { type: ActionTypes.ChangeFormStep, callback: formApi.changeStep },
  ]
</script>

<Provider {actions} data={dataContext}>
  <div use:styleable={$component.styles} class={size}>
    <slot />
  </div>
</Provider>
