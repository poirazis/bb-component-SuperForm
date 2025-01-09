<script>
  import { getContext, onMount, onDestroy, setContext } from "svelte";
  import RowFormWrapper from "./RowFormWrapper.svelte";
  import SuperButton from "../../bb_super_components_shared/src/lib/SuperButton/SuperButton.svelte";
  import { fly } from "svelte/transition";
  import { get } from "svelte/store";

  const FieldTypeToComponentMap = {
    string: "plugin/bb-component-SuperFieldText",
    number: "plugin/bb-component-SuperFieldNumber",
    bigint: "plugin/bb-component-SuperFieldNumber",
    options: "plugin/bb-component-SuperFieldOption",
    array: "plugin/bb-component-SuperFieldOptions",
    jsonarray: "plugin/bb-component-SuperFieldArray",
    boolean: "plugin/bb-component-SuperFieldBoolean",
    longform: "longformfield",
    datetime: "plugin/bb-component-SuperFieldDatetime",
    link: "plugin/bb-component-SuperFieldRelationship",
    json: "jsonfield",
    barcodeqr: "codescanner",
    bb_reference_single: "plugin/bb-component-SuperFieldRelationship",
    bb_reference: "plugin/bb-component-SuperFieldRelationship",
    signature_single: "signaturesinglefield",
    attachment_single: "plugin/bb-component-SuperFieldAttachment",
    attachment: "plugin/bb-component-SuperFieldAttachmentList",
  };

  const {
    Block,
    BlockComponent,
    memo,
    Provider,
    ContextScopes,
    API,
    builderStore,
    notificationStore,
    fetchDatasourceSchema,
    enrichButtonActions,
    fetchData,
    QueryUtils,
    processStringSync,
    ActionTypes,
  } = getContext("sdk");

  const component = getContext("component");
  const allContext = getContext("context");
  const outerForm = getContext("form");
  const outerFormStep = getContext("form-step");
  const outerFormJson = getContext("super-form-json");
  const outerFormEmbed = getContext("super-form-embed");
  const outerFormSettings = getContext("super-form-settings");
  const outerFormApi = getContext("super-form-api");

  export let formTitle;
  export let titleIcon;
  export let formSubtitle;
  export let nested;
  export let subformType = "link";
  export let extendActions;
  export let formButtons = [];
  export let hideDefaultActions = false;
  export let actionType = "create";
  export let dataSource;

  export let disabled;
  export let fields;
  export let canEdit;
  export let canDelete;
  export let mode = "formInput";
  export let relViewMode = "text";
  export let optionsViewMode = "pills";
  export let showDirty = true;
  export let formColumns = 3;

  // Single Row Settings
  export let rowId;

  // Multi Row Settings
  export let multiRow = false;
  export let actionsMode;
  export let rowActionButtons;
  export let minRows = 1;
  export let maxRows;
  export let compact;
  export let canAdd = true;
  export let data;

  // Multi Step Settings
  export let multiStep;
  export let steps = [];

  // Sub Form Settings
  export let inStep = 1;
  export let colSpan = 6;
  export let rowSpan = 1;
  export let outerField;
  export let outerLinkField;
  export let quiet;
  export let hideIfTrue = false;

  // Data Settings
  export let filter;
  export let limit = 10;

  // Events
  export let afterSave;
  export let afterDelete;

  // Styling
  export let labelPosition;
  export let labelTemplate;
  export let beautifyLabels = true;
  export let slotFirst;
  export let entitySingular;
  export let entityPlural;

  // Internal Properties
  let _outerField;
  let _outerLinkField;
  let _subFormType;
  let _multiRow = multiRow;

  let mounted;
  let dirty;

  let jsonFields = memo([]);
  let embedFields = memo([]);
  $: jsonFields.set(getJSONFields(schema));
  $: embedFields.set(getEmbedFields(schema));

  // When acting as a subform
  let outerFormField;
  let outerFieldState;
  let outerFieldApi;
  let outerFieldSchema;
  let outerFieldValue;

  let unsubscribe;

  let loaded;
  let empty;
  let emptyData;
  let definition;
  let schema;
  let primaryDisplay;
  let rows = [{}];
  let existingRows = [];
  let titleHover;
  let viewToEdit;
  let activeStep = 0;

  let newRowForm = [];
  let existingRowForm = [];

  let richFormSubtitle;

  // Initialize Stores
  const comp_id = $component.id;
  const dataSourceStore = memo(dataSource);
  const formSettings = memo({});
  $: dataSourceStore.set(dataSource);
  $: formSettings.set({
    $dataSourceStore,
    actionType,
    multiStep,
  });

  $: _actionType = $outerFormSettings?.actionType ?? actionType;
  // Trigger Reinitialization
  $: superFormApi.init($dataSourceStore, multiRow, minRows, _actionType);

  // Maintain Step while in Builder
  $: inBuilder = $builderStore.inBuilder;
  $: isParent = superFormApi?.subforms?.length;
  $: activeStep = $builderStore.metadata?.step || 0;
  $: if (inBuilder && multiStep) {
    existingRowForm.forEach((x) => x?.formApi?.setStep(activeStep + 1));
    newRowForm.forEach((x) => x?.formApi?.setStep(activeStep + 1));
  }

  // Keep schema uptodate with Form role and field changes
  $: superFormApi?.fetchSchema($dataSourceStore, outerField, outerLinkField);

  // Enrich Form Steps
  $: richSteps = superFormApi.enrichSteps(steps, schema, multiStep);

  $: singleFilter = {
    logicalOperator: "all",
    onEmptyFilter: "none",
    groups: [
      {
        logicalOperator: "any",
        filters: [
          {
            valueType: "Value",
            field: "_id",
            type: "string",
            operator: "equal",
            noValue: false,
            value: rowId,
          },
        ],
      },
    ],
  };

  // Fetch and load initial data
  // If in Builder and no rowid has been set, fetch the first row for preview, else apply
  $: query =
    multiRow || (!multiRow && inBuilder && !rowId)
      ? QueryUtils.buildQuery(filter)
      : QueryUtils.buildQuery(singleFilter);

  $: fetch = superFormApi.createFetch($dataSourceStore, multiRow);
  $: fetch?.update({ query });
  $: superFormApi.getInitialData($fetch, multiRow, _actionType);

  $: empty =
    loaded && richSteps[0]?.fields?.length < 1 && $component.children < 1;

  $: dirty = rows.filter((row) => row._isDirty).length > 0;

  $: valid = rows.filter((row) => row._isValid == false).length < 1;

  // If we are a nested form of a multi step form and designated to appear on a specific step.
  $: visible =
    hideIfTrue !== true &&
    hideIfTrue !== "true" &&
    ($outerFormSettings?.multiStep && outerFormStep
      ? $outerFormStep == inStep
      : true);

  // Data Loading Management
  $: batchMode =
    actionsMode == "batch" && _actionType != "view" && !outerLinkField;
  $: eventMode = actionsMode == "individual";

  $: canAddRows =
    (_actionType == "update" && canAdd && multiRow) ||
    _actionType == "create" ||
    outerLinkField;

  $: enrichedFormTitle = superFormApi.enrichFormTitle(
    formTitle,
    formSubtitle,
    $formSettings,
    loaded,
    emptyData,
    $fetch
  );

  /** The Super Form API
   *
   *
   */
  const superFormApi = {
    id: comp_id,
    get __dirty() {
      return rows.filter((row) => row._isDirty).length > 0;
    },
    init: () => {
      viewToEdit = false;
      existingRows = [];
      rows =
        _actionType == "create"
          ? multiRow
            ? new Array(minRows || 1).fill({})
            : [{}]
          : [];

      console.log("Init", rows, existingRows);
    },
    async fetchSchema(dataSource) {
      schema = undefined;
      loaded = false;

      if (nested && subformType == "link" && dataSource?.type !== "link") {
        schema = {};
        loaded = true;
        return;
      }

      if (dataSource?.name != "Custom" || subformType != "json") {
        if (dataSource.schema || dataSource.type == "viewV2") {
          schema = await fetchDatasourceSchema(dataSource);
        } else if (dataSource?.tableId) {
          try {
            definition = await API.fetchTableDefinition(dataSource.tableId);
            schema = definition?.schema;
            primaryDisplay = definition.primaryDisplay;
            _outerLinkField = dataSource?.fieldName;
          } catch (error) {
            definition = null;
            schema = null;
          }
        } else {
          schema = await fetchDatasourceSchema(dataSource);
        }
      }

      if (_outerLinkField || outerField)
        await superFormApi.fetchSchemaForField();
      else if (_actionType == "create") loaded = true;
    },
    fetchSchemaForField: async (field) => {
      // if we are dealing with a field, get schema from parent form field definition
      if (nested && (outerField || outerLinkField)) {
        if (outerFormField) {
          outerFieldApi.deregister();
          unsubscribe();
        }

        outerFormField = outerForm?.formApi?.registerField(
          outerLinkField || outerField,
          outerLinkField ? "link" : "json",
          null,
          false,
          false,
          null,
          1
        );

        unsubscribe = outerFormField?.subscribe((value) => {
          outerFieldState = value?.fieldState;
          outerFieldApi = value?.fieldApi;
          outerFieldSchema = value?.fieldSchema;
          outerFieldValue = value?.fieldState?.value;
        });

        if (outerFieldSchema?.type == "json") {
          if (subformType == "json") {
            schema = {};
            schema[outerField] = outerFieldSchema;

            if (_actionType != "create") {
              existingRows[0] = {
                [outerField]: {
                  ...outerFieldValue,
                  type: "row",
                  tableId: outerForm?.dataSource?.tableId,
                },
              };
            }
          } else {
            if (_actionType != "create") {
              console.log(outerFieldValue);
              existingRows[0] = {
                ...outerFieldValue,
                type: "row",
                tableId: dataSource.tableId,
              };
            }
          }

          emptyData = _actionType != "create" && existingRows?.length < 1;
        } else if (outerFieldSchema?.type == "link") {
          _multiRow =
            outerFieldSchema.relationshipType == "many-to-many" &&
            multiRow !== false;
        }

        loaded = true;
      }
    },
    enrichSteps: (steps) => {
      if (multiStep && steps)
        return steps.map((step, index) => {
          return {
            title: step.title
              ? processStringSync(step.title, $allContext)
              : "Step " + (index + 1),
            desc: step.desc ? processStringSync(step.desc, $allContext) : null,
            fields: superFormApi.getDefaultFields(step.fields, schema),
            buttons: superFormApi.enrichStepButtons(
              step.buttons,
              index,
              steps.length
            ),
          };
        });
      else
        return [
          {
            title: null,
            fields: superFormApi.getDefaultFields(fields, schema),
            buttons: [],
          },
        ];
    },
    enrichStepButtons: (buttons, index, total) => {
      const defaultButtonsFirst = [
        {
          text: "Next",
          type: "primary",
          size: "M",
          onClick: [
            {
              "##eventHandlerType": "Validate Form",
              parameters: {
                componentId: comp_id,
              },
            },
            {
              parameters: {
                type: "next",
                componentId: comp_id,
              },
              "##eventHandlerType": "Change Form Step",
            },
          ],
        },
      ];
      const defaultButtonsMiddle = [
        {
          text: "Back",
          type: "secondary",
          size: "M",
          onClick: [
            {
              parameters: {
                type: "prev",
                componentId: comp_id,
              },
              "##eventHandlerType": "Change Form Step",
            },
          ],
        },
        {
          text: "Next",
          type: "primary",
          size: "M",
          onClick: [
            {
              "##eventHandlerType": "Validate Form",
              parameters: {
                componentId: comp_id,
              },
            },
            {
              parameters: {
                type: "next",
                componentId: comp_id,
              },
              "##eventHandlerType": "Change Form Step",
            },
          ],
        },
      ];
      const defaultButtonsLast = [
        {
          text: "Back",
          type: "secondary",
          size: "M",
          onClick: [
            {
              parameters: {
                type: "prev",
                componentId: comp_id,
              },
              "##eventHandlerType": "Change Form Step",
            },
          ],
        },
        {
          text: "Save",
          type: "cta",
          size: "M",
          onClick: [
            {
              "##eventHandlerType": "Validate Form",
              parameters: {
                componentId: comp_id,
              },
            },
            {
              parameters: {
                confirm: true,
                customTitleText: entitySingular
                  ? "Creating New " + entitySingular
                  : "Creating New " + $dataSourceStore.label,
                confirmText:
                  "Are you sure you want to save this " +
                  (entitySingular || $dataSourceStore.label) +
                  " ?",
                confirmButtonText: "Confirm",
                cancelButtonText: "Cancel",
              },
              "##eventHandlerType": "Prompt User",
            },
            {
              "##eventHandlerType": "Save Row",
              parameters: {
                providerId: comp_id + "-form",
                tableId: $dataSourceStore.tableId,
              },
            },
          ],
        },
      ];

      if (buttons) return buttons;
      else if (total == index + 1 || total == 1) return defaultButtonsLast;
      else if (index > 0 && index < total) return defaultButtonsMiddle;
      else return defaultButtonsFirst;
    },
    getDefaultFields: (fields, schema) => {
      if (!schema) {
        return [];
      }

      let defaultFields = [];
      if (!fields) {
        fields = [];
        Object.values(schema)
          .filter(
            (field) =>
              !field.autocolumn && field.visible != false && !field.nestedJSON
          )
          .sort((a, b) => {
            return a.order - b.order;
          })
          .forEach((field) => {
            if (field?.type == "json") {
              if (!field.schema) {
                defaultFields.push({
                  label: field.name,
                  field: field.name,
                  active: true,
                });
              } else {
                Object.keys(field.schema ?? {})?.forEach((sub_field) => {
                  if (field.schema[sub_field].type != "json")
                    if (field.schema[sub_field].type == "array" && !nested) {
                      return;
                    } else
                      defaultFields.push({
                        label: outerField
                          ? sub_field
                          : field.name + "." + sub_field,
                        field: field.name + "." + sub_field,
                        active: true,
                      });
                });
              }
            } else {
              defaultFields.push({
                field: field.name,
                label: field.name.endsWith("_self_")
                  ? field.name.slice(0, -6)
                  : field?.name || field.field,
                active: true,
              });
            }
          });
      }

      return [...fields, ...defaultFields]
        .filter((field) => field.active)
        .map((field) => {
          return {
            ...field,
            label:
              beautifyLabels && !labelTemplate
                ? beautifyLabel(field.label)
                : field.label,
          };
        });
    },
    createFetch: (datasource) => {
      return fetchData({
        API,
        datasource,
        options: {
          limit: outerLinkField ? 1000 : multiRow ? limit : 1,
        },
      });
    },
    getInitialData: (res) => {
      if (res?.loading) return;

      if (outerField || _actionType == "create") {
        loaded = true;
        emptyData = false;
        return;
      }

      if (multiRow && data) {
        existingRows = safeParse(data);
      } else {
        existingRows = res?.rows ?? [];
        if (outerLinkField && res?.rows.length < 1 && _actionType == "update")
          rows = [{}];
        else rows = [];
      }

      emptyData =
        _actionType != "create" && existingRows?.length < 1 && !outerLinkField;

      loaded = true;
    },
    enrichFormTitle: (titleTemplate) => {
      if (!loaded || !_actionType) return;

      let activeRow = existingRows?.length ? existingRows[0] : {};
      let context = {
        ...$allContext,
        [comp_id]: { ...activeRow },
      };

      let actionName = _actionType[0].toUpperCase() + _actionType.substr(1);
      let title;

      if (nested) {
        title = titleTemplate
          ? processStringSync(titleTemplate, context)
          : beautifyLabels
            ? beautifyLabel(
                outerField || outerLinkField || $dataSourceStore.fieldName
              )
            : outerField || outerLinkField || $dataSourceStore.fieldName;
      } else {
        title = titleTemplate
          ? processStringSync(titleTemplate, context)
          : _actionType != "create" && !multiRow
            ? activeRow?.[primaryDisplay]
              ? activeRow[primaryDisplay]
              : $dataSourceStore.label
            : actionName +
              " " +
              (multiRow
                ? entityPlural || beautifyLabel($dataSourceStore.label)
                : entitySingular || beautifyLabel($dataSourceStore.label));

        richFormSubtitle = formSubtitle
          ? processStringSync(formSubtitle, context)
          : undefined;
      }

      return title;
    },
    saveRow: async () => {
      if (!superFormApi.validate()) return;
      // Have child Super Forms save themselves first
      for (let i = 0; i < subforms.length; i++) {
        await subforms[i]?.saveRow();
      }

      // We are managing a relationship field
      if (outerLinkField && _actionType == "create") {
        if (multiRow) {
          let newRows = [];
          for (let i = 0; i < rows.length; i++) {
            let formState = newRowForm[i]?.formState;
            let newRow = get(formState)?.values;

            const res = await API.saveRow({
              ...newRow,
              tableId: $dataSourceStore.tableId,
            });

            if (res) newRows.push(res["_id"]);
          }

          // Update parent form field value to the newly created rows
          outerFieldApi.setValue(newRows);
        } else {
          let formState = newRowForm[0]?.formState;
          let newRow = get(formState)?.values;

          const res = await API.saveRow({
            ...newRow,
            tableId: $dataSourceStore.tableId,
          });

          if (res) outerFieldApi.setValue([res["_id"]]);
        }
      } else if (outerLinkField) {
        if (multiRow) {
          for (let i = 0; i < existingRows.length; i++) {
            let formState = existingRowForm[i]?.formState;
            let row = {
              _id: existingRows[i]["_id"],
              tableId: existingRows[i]["tableId"],
              ...unflattenObject(get(formState)?.values ?? {}),
            };

            const res = await API.patchRow(row);
          }
        } else {
          let formState = existingRowForm[0]?.formState;
          let row = {
            _id: existingRows[0]["_id"],
            tableId: existingRows[0]["tableId"],
            ...unflattenObject(get(formState)?.values ?? {}),
          };

          const res = await API.patchRow({
            ...row,
          });
          if (res) outerFieldApi.setValue([res["_id"]]);
        }
      }

      // We are managing a JSON fiel
      if (outerField && visible) {
        let formState = existingRowForm[0]?.formState;
        if (_actionType == "create") formState = newRowForm[0].formState;
        let object = unflattenObject(get(formState)?.values);

        if (subformType == "embed") outerFieldApi.setValue(object);
        else outerFieldApi.setValue(object[outerField]);
      }
    },
    validate: () => {
      let res = true;

      // Validate all sub Super Forms
      for (let i = 0; i < subforms.length; i++) {
        res = res && subforms[i]?.validate();
      }

      // Validate all new row forms
      for (let i = 0; i < newRowForm.length; i++) {
        res = res && newRowForm[i]?.formApi?.validate();
      }

      // Validate all existing row forms
      for (let i = 0; i < existingRowForm.length; i++) {
        res = res && existingRowForm[i]?.formApi?.validate();
      }
      return res;
    },
    reset: () => {
      for (let i = 0; i < subforms.length; i++) {
        subforms[i]?.reset();
      }

      for (let i = 0; i < newRowForm.length; i++) {
        newRowForm[i]?.formApi?.reset();
        rows[i] = {};
      }
    },
    registerSubform: (subform) => {
      subforms.push(subform);
    },
    unregisterSubform: (subform) => {
      let idx = subforms.findIndex((x) => x.id == subform.id);
      if (idx > -1) subforms = subforms.splice(idx, 1);
    },
  };
  // The array of registered subform instances
  let subforms = [];

  const actions = [
    {
      type: ActionTypes.ValidateForm,
      callback: superFormApi.validate,
    },
    { type: ActionTypes.ClearForm, callback: superFormApi.reset },
    { type: ActionTypes.UpdateFieldValue, callback: () => {} },
    {
      type: ActionTypes.ChangeFormStep,
      callback: (e) => {
        newRowForm[0]?.formApi?.changeStep(e);
      },
    },
    { type: ActionTypes.ScrollTo, callback: () => {} },
  ];

  const getComponentForField = (field) => {
    const fieldSchemaName = field.field || field.name;
    let json_path;
    let innerType;
    let fieldSchema;

    if (fieldSchemaName.includes(".")) {
      json_path = fieldSchemaName.split(".");
      fieldSchema = schema[json_path[0]]?.schema;
      innerType = fieldSchema[json_path[1]]?.type;
      if (innerType == "array") innerType = "jsonarray";
    } else {
      innerType = schema[fieldSchemaName]?.type;
      if (innerType == "array") {
        if (schema[fieldSchemaName].constraints.inclusion.length < 1)
          innerType = "jsonarray";
      }
    }

    if (!fieldSchemaName) {
      return null;
    }

    const type = fieldSchemaName.endsWith("_self_") ? "link" : innerType;

    return FieldTypeToComponentMap[type];
  };

  const getPropsForField = (field, idx) => {
    return {
      ...field,
      field: field?.field,
      label: labelTemplate
        ? processStringSync(labelTemplate, { field: field?.field })
        : field?.label,
      placeholder: field.placeholder || field.label || field.field,
      useOptionColors: true,
      readonly: field?.readonly || _actionType == "view",
      autocomplete: field?.autocomplete,
      search: true,
      role: mode,
      relViewMode,
      optionsViewMode,
      compact: true,
      showDirty,
      labelPosition: compact && idx > 0 ? false : labelPosition,
      ownId: field?.field?.includes("_self_")
        ? existingRows[idx]?.["_id"]
        : null,
      direction: field?.direction == "vertical" ? "column" : "row",
      controlType: "select",
    };
  };

  const beautifyLabel = (label) => {
    if (!beautifyLabels || !label) return label;

    let fields = label.split(".");
    fields.forEach((field, index) => {
      let words = field.split("_");
      words.forEach((word, index) => {
        words[index] = word[0]?.toUpperCase() + word?.slice(1);
      });
      fields[index] = words.join(" ");
    });
    return fields.join(" - ");
  };

  const safeParse = (data) => {
    try {
      return JSON.parse(data);
    } catch (ex) {
      return [];
    }
  };

  // Single Row action handlers
  const handleSave = async () => {
    let cmd;
    let cmd_after;
    let event;
    let promptUserAction = [
      {
        parameters: {
          confirm: true,
          customTitleText: entitySingular
            ? "Creating New " + entitySingular
            : "Creating New " + $dataSourceStore.label,
          confirmText:
            "Are you sure you want to save this " +
            (entitySingular || $dataSourceStore.label) +
            " ?",
          confirmButtonText: "Confirm",
          cancelButtonText: "Cancel",
        },
        "##eventHandlerType": "Prompt User",
      },
    ];

    if (superFormApi.validate()) {
      if (_actionType == "create") {
        let promptAction = enrichButtonActions(promptUserAction, $allContext);
        let response = await promptAction();

        if (response) {
          await superFormApi.saveRow();
          event = [
            {
              parameters: {
                confirm: false,
                notificationOverride: null,
                tableId: $dataSourceStore.tableId,
                providerId: comp_id + "-form",
              },
              "##eventHandlerType": "Save Row",
            },
          ];

          cmd = enrichButtonActions(event, $allContext);
          cmd_after = enrichButtonActions(afterSave, $allContext);

          await cmd?.();
          await cmd_after?.();

          superFormApi.reset();
        }
      } else {
        await superFormApi.saveRow();
        let formState = existingRowForm[0].formState;
        let richValues = {
          ...existingRows[0],
          ...unflattenObject(get(formState).values),
        };

        event = [
          {
            parameters: {
              confirm: true,
              confirm: true,
              customTitleText: entitySingular
                ? "Saving " + entitySingular + " " + enrichedFormTitle
                : "Saving " + $dataSourceStore.label + " " + enrichedFormTitle,
              confirmText:
                "Are you sure you want to save this " +
                (entitySingular || $dataSourceStore.label) +
                " ?",
              confirmButtonText: "Confirm",
              cancelButtonText: "Cancel",
              tableId: $dataSourceStore.tableId,
              providerId: comp_id + "-form",
            },
            "##eventHandlerType": "Save Row",
          },
        ];

        // Expose the internal Form to the outer context
        cmd = enrichButtonActions(event, {
          ...$allContext,
          [comp_id + "-form"]: richValues,
        });
        cmd_after = enrichButtonActions(afterSave, $allContext);
        await cmd?.();
        await cmd_after?.();

        if (viewToEdit) {
          viewToEdit = false;
          _actionType = "view";
        }

        fetch?.refresh();
      }
    }
  };

  const handleCustomAction = async (action) => {
    let formState = existingRowForm[0]?.formState;
    let richValues = unflattenObject(get(formState)?.values);

    // Expose the internal form in case of update
    let cmd =
      _actionType != "create"
        ? enrichButtonActions(action, {
            ...$allContext,
            [comp_id + "-form"]: {
              ...existingRows[0],
              ...richValues,
            },
          })
        : enrichButtonActions(action, $allContext);

    await cmd?.();
  };

  const handleSaveMany = async () => {
    if (superFormApi.validate()) {
      if (_actionType == "create") {
        for (let i = 0; i < rows.length; i++) {
          const res = await API.saveRow({
            ...rows[i],
            tableId: $dataSourceStore.tableId,
          });

          if (res) {
            newRowForm[i].formApi.reset();
          }

          notificationStore.actions.success(
            rows.length + " " + entityPlural + " Saved"
          );
        }
        rows = [{}];
      } else {
        for (let i = 0; i < existingRows.length; i++) {
          const formState = existingRowForm[i].formState;
          const values = unflattenObject(get(formState).values);
          const res = await API.saveRow({
            ...existingRows[i],
            ...values,
          });
        }
      }

      // Enrich and execute afterSave events
      let cmd_after = enrichButtonActions(afterSave, $allContext);
      await cmd_after?.();

      fetch?.refresh?.();
    }
  };

  const handleDelete = async (index, type) => {
    if (type == "new") {
      if (rows.length == minRows && _actionType != "update") {
        notificationStore.actions.warning("Minimum " + minRows + " rows");
      } else {
        rows.splice(index, 1);
        rows = rows;
      }
    } else {
      if (multiRow && batchMode) {
        existingRows[index]["_deleted"] = !existingRows[index]["_deleted"];
        // pendingChanges = true;
      } else if (eventMode) {
        let context = {
          ...$allContext,
          [comp_id]: { ...existingRows[index] },
        };
        let cmd = enrichButtonActions(onRowDelete, context);
        await cmd();
        existingRows.splice(index, 1);
      } else {
        let event = [
          {
            parameters: {
              confirm: true,
              notificationOverride: null,
              tableId: $dataSourceStore.tableId,
              rowId: existingRows[index]["_id"],
            },
            "##eventHandlerType": "Delete Row",
          },
        ];
        let cmd = enrichButtonActions(event, $allContext);
        let after_cmd = enrichButtonActions(afterDelete, $allContext);
        await cmd?.();
        await after_cmd?.();
      }
    }
  };

  const handleAddRow = () => {
    let last = rows.length - 1;

    if (last < 0 || newRowForm[last].formApi.validate()) {
      if (maxRows && rows.length == maxRows) {
        notificationStore.actions.warning("Maximum " + maxRows + " rows");
      } else {
        rows.push({});
        rows = rows;
      }
    }
  };

  const unflattenObject = (obj, delimiter = ".") => {
    if (obj) {
      return Object.keys(obj).reduce((res, k) => {
        k.split(delimiter).reduce(
          (acc, e, i, keys) =>
            acc[e] ||
            (acc[e] = isNaN(Number(keys[i + 1]))
              ? keys.length - 1 === i
                ? obj[k]
                : {}
              : []),
          res
        );
        return res;
      }, {});
    }
  };

  /** Propagate changes to the outform when the form values change */
  const handleValuesChanges = (index, values) => {
    if (outerFieldApi) {
      if (outerFieldSchema.type == "json") {
        if (_actionType == "create") {
          if (!multiRow) {
            if (subformType == "json") {
              outerFieldApi.setValue(values[outerField]);
            } else outerFieldApi.setValue(values);
          } else {
            let cleanRows = rows.map((x) => {
              delete x._isDirty;
              return x;
            });
            outerFieldApi.setValue({ rows: cleanRows });
          }
        } else {
          outerFieldApi.setValue(values[outerField]);
        }
      }
    }
  };

  $: if (inBuilder && $component.selected && outerFormSettings && !nested) {
    builderStore.actions.updateProp("nested", true);
  }

  $: if (
    inBuilder &&
    $component.selected &&
    !outerFormSettings &&
    nested &&
    mounted
  ) {
    builderStore.actions.updateProp("nested", false);
    builderStore.actions.updateProp("_instanceName", "New Super Form");
  }

  $: if (inBuilder && $component.selected && nested && _outerField) {
    builderStore.actions.updateProp("subformType", _subFormType);
    builderStore.actions.updateProp("outerField", _outerField);
    builderStore.actions.updateProp(
      "_instanceName",
      "Subform JSON " +
        (subformType == "embed" ? "Embed - " : "Column - ") +
        _outerField
    );
    _outerField = undefined;
  }

  $: if (inBuilder && $component.selected && nested && _outerLinkField) {
    builderStore.actions.updateProp("outerLinkField", _outerLinkField);
    builderStore.actions.updateProp(
      "_instanceName",
      "Subform Link - " + _outerLinkField
    );
    _outerLinkField = undefined;
  }

  $: if (inBuilder && $component.selected && nested && _multiRow) {
    builderStore.actions.updateProp("multiRow", true);
    _multiRow = undefined;
  }

  $: $component.styles = {
    ...$component.styles,
    normal: {
      ...$component.styles.normal,
      "grid-column": "span " + colSpan * 6,
      "grid-row": "span " + rowSpan,
      display: "flex",
      "align-items": "stretch",
      display: visible ? "flex" : "none",
    },
  };

  const getJSONFields = (schema) => {
    let jsonFields = [];
    if (schema)
      for (let field in schema) {
        if (schema[field].type == "json")
          if (schema[field].schema) jsonFields.push(field);
      }

    return jsonFields;
  };

  const getEmbedFields = (schema) => {
    let embedFields = [];
    if (schema)
      for (let field in schema) {
        if (schema[field].type == "json")
          if (!schema[field].schema) embedFields.push(field);
      }

    return embedFields;
  };

  // Expose Context for Sub forms to adjust their Action type
  setContext("super-form-json", jsonFields);
  setContext("super-form-embed", embedFields);
  setContext("super-form-settings", formSettings);
  setContext("super-form-api", superFormApi);
  $: setContext("field-group", labelPosition);
  $: setContext("field-group-columns", formColumns);

  onMount(() => {
    if (outerFormApi) {
      outerFormApi.registerSubform(superFormApi);
    }
    mounted = true;
  });

  onDestroy(() => {
    superFormApi?.saveRow();
    outerFormApi?.unregisterSubform(superFormApi);
  });
</script>

<!-- svelte-ignore a11y-no-static-element-interactions -->
<!-- svelte-ignore a11y-click-events-have-key-events -->
<!-- svelte-ignore a11y-mouse-events-have-key-events -->

<Block>
  <Provider
    {actions}
    data={_actionType != "create"
      ? {
          ...existingRows[0],
        }
      : {
          ...rows[0],
          __value: rows[0],
          __currentStep: activeStep,
          __currentStepValid: valid,
          __valid: valid,
          __dirty: dirty,
        }}
  />
  {#if loaded && !empty && !emptyData}
    {#if richSteps[0]?.fields}
      <div class="super-form" class:form-field={outerForm} class:quiet>
        {#if enrichedFormTitle?.trim?.()}
          <div class="form-header">
            <div
              class="form-title"
              style:cursor={canEdit ? "pointer" : null}
              on:mouseover={() => (titleHover = canEdit)}
              on:mouseleave={() => (titleHover = false)}
              on:click|self={canEdit
                ? () => {
                    _actionType = viewToEdit ? "view" : "update";
                    viewToEdit = !viewToEdit;
                  }
                : () => {}}
            >
              <span class="title" class:small={formSubtitle}>
                {#if titleIcon || superFormApi.__dirty}
                  <i class={titleIcon || "ri-edit-line"} />
                {/if}
                {nested ? enrichedFormTitle.toUpperCase() : enrichedFormTitle}
                {#if canEdit && titleHover && _actionType == "view"}
                  <i class={"ri-pencil-fill"} style="font-size: 16px;" />
                {/if}
                {#if viewToEdit}
                  <i class={"ri-arrow-go-back-fill"} style="font-size: 16px;" />
                {/if}
              </span>

              {#if richFormSubtitle}
                <span class="sub-title">{richFormSubtitle}</span>
              {/if}
            </div>
            <div class="form-buttons">
              {#if extendActions && formButtons.length}
                {#each formButtons as { type, text, icon, onClick, quiet }}
                  <SuperButton
                    {text}
                    {type}
                    {icon}
                    {quiet}
                    {disabled}
                    on:click={handleCustomAction(onClick)}
                  />
                {/each}
              {/if}

              {#if !nested && $dataSourceStore?.tableId}
                {#if !hideDefaultActions && _actionType != "view"}
                  {#if multiStep && _actionType == "create"}
                    {#if richSteps[activeStep]?.buttons}
                      {#each richSteps[activeStep]?.buttons as button}
                        <SuperButton
                          {...button}
                          on:click={button.text == "Save" && isParent
                            ? handleSave
                            : () => {}}
                          onClick={button.text == "Save" && isParent
                            ? () => {}
                            : enrichButtonActions(button.onClick, $allContext)}
                        />
                      {/each}
                    {/if}
                  {:else}
                    {#if !multiRow && _actionType != "create" && canDelete}
                      <SuperButton
                        icon="ri-delete-bin-line"
                        type="warning"
                        quiet
                        text="Delete"
                        on:click={() => handleDelete(0)}
                      />
                    {/if}
                    {#if multiRow && actionsMode == "batch"}
                      <SuperButton
                        icon="ri-save-line"
                        text="Save All"
                        type="primary"
                        {disabled}
                        on:click={handleSaveMany}
                      />
                    {:else if !multiRow}
                      {#if _actionType == "create"}
                        <SuperButton
                          text="Reset"
                          type="secondary"
                          quiet
                          {disabled}
                          on:click={superFormApi.reset}
                        />
                      {/if}
                      <SuperButton
                        icon="ri-save-line"
                        text="Save"
                        {disabled}
                        type={"secondary"}
                        on:click={handleSave}
                      />
                    {/if}
                  {/if}
                {/if}
              {/if}
            </div>
          </div>
        {/if}

        <div class="form-body">
          <!-- Existing Rows -->
          {#each existingRows as row, index}
            <div class="form-row">
              <Provider data={{ ...row, index }} scope={ContextScopes.Local}>
                <BlockComponent
                  type="form"
                  context="form"
                  props={{
                    actionType: "Update",
                    dataSource:
                      outerField && subformType == "json"
                        ? {
                            type: "table",
                            tableId: outerForm?.dataSource?.tableId,
                          }
                        : { ...$dataSourceStore, type: "table" },
                    disabled: row["_deleted"] || disabled,
                    readonly: !disabled && _actionType === "view",
                  }}
                  styles={{
                    normal: {
                      width: "100%",
                    },
                  }}
                >
                  <RowFormWrapper
                    bind:formObject={existingRowForm[index]}
                    {row}
                    {comp_id}
                    {disabled}
                    {multiRow}
                    {richSteps}
                    {actionsMode}
                    {rowActionButtons}
                    {compact}
                    {quiet}
                    inView={_actionType == "view"}
                    canAdd={index == existingRows.length - 1 &&
                      rows.length < 1 &&
                      canAdd}
                    {batchMode}
                    {canDelete}
                    on:delete={(e) => handleDelete(index)}
                    on:save={(e) => handleSave(index)}
                    on:add-row={handleAddRow}
                    on:valuesChanged={(e) => {
                      dirty = true;
                      // handleValuesChanges(index, e.detail);
                    }}
                  >
                    {#each richSteps as step, idx}
                      <BlockComponent
                        type="formstep"
                        name={"Form Setp - " + (idx + 1)}
                        props={{
                          step: idx + 1,
                          _instanceName: `Step ${idx + 1}`,
                        }}
                      >
                        {#if richSteps[idx]?.desc}
                          <div class="step-desc">
                            {richSteps[idx].desc}
                          </div>
                        {/if}
                        <div
                          class="form-grid"
                          class:labels-left={labelPosition == "left"}
                          style:--form-columns={formColumns * 6}
                        >
                          {#if slotFirst}
                            <slot />
                          {/if}
                          {#each step.fields as field, idx}
                            {#if getComponentForField(field) && field.active}
                              <BlockComponent
                                context={"field_" + idx}
                                type={getComponentForField(field)}
                                props={getPropsForField(field, index)}
                                order={idx}
                                interactive
                                name={"SuperField-" + field?.field}
                              />
                            {/if}
                          {/each}
                          {#if !slotFirst}
                            <slot />
                          {/if}
                        </div>
                      </BlockComponent>
                    {/each}
                  </RowFormWrapper>
                </BlockComponent>
              </Provider>
            </div>
          {/each}

          <!-- New Rows -->
          {#if rows.length}
            {#each rows as row, index}
              <div
                class="form-row"
                transition:fly={index > 0
                  ? { duration: 130, y: -40 }
                  : { duration: 0 }}
              >
                <BlockComponent
                  type="form"
                  context="form"
                  props={{
                    actionType: "Create",
                    dataSource: $dataSourceStore,
                    disabled,
                  }}
                  styles={{
                    normal: {
                      width: "100%",
                    },
                  }}
                >
                  <RowFormWrapper
                    bind:formObject={newRowForm[index]}
                    {row}
                    isNew
                    {compact}
                    {quiet}
                    {disabled}
                    canDelete={index > 0 ||
                      rows.length > 1 ||
                      existingRows.length}
                    canAdd={index == rows.length - 1}
                    {rowActionButtons}
                    {multiRow}
                    {richSteps}
                    actionsMode={nested ? "batch" : actionsMode}
                    on:step-change={(e) => (activeStep = e.detail - 1)}
                    on:delete={(e) => handleDelete(index, "new")}
                    on:save={(e) => handleSave(index, "new")}
                    on:add-row={handleAddRow}
                    on:valuesChanged={(e) => {
                      rows[index] = {
                        ...rows[index],
                        ...e.detail,
                      };
                      handleValuesChanges(index, e.detail);
                    }}
                  >
                    {#each richSteps as step, idx}
                      <BlockComponent
                        type="formstep"
                        name={"Form Setp - " + (idx + 1)}
                        props={{
                          step: idx + 1,
                          _instanceName: `Step ${idx + 1}`,
                        }}
                      >
                        {#if richSteps[activeStep]?.desc}
                          <div class="step-desc">
                            {richSteps[activeStep].desc}
                          </div>
                        {/if}
                        <div
                          class="form-grid"
                          class:labels-left={labelPosition == "left"}
                          style:--form-columns={formColumns * 6}
                        >
                          {#if slotFirst}
                            <slot />
                          {/if}
                          {#each step.fields as field}
                            {#if getComponentForField(field) && field.active}
                              <BlockComponent
                                type={getComponentForField(field)}
                                props={getPropsForField(field, index)}
                                name={"SuperField-" + field?.field}
                              />
                            {/if}
                          {/each}
                          {#if !slotFirst}
                            <slot />
                          {/if}
                        </div>
                      </BlockComponent>
                    {/each}
                  </RowFormWrapper>
                </BlockComponent>
              </div>
            {/each}
          {/if}
        </div>
      </div>
    {/if}
  {/if}

  {#if loaded && empty && inBuilder}
    <div class="empty">
      {#if !nested}
        <h3>Welcome to the Super Form</h3>
        <div class="message">
          <i class="ri-information-line" />
          Please add some Fields to get Started
        </div>
      {:else}
        <h3>
          Subform Type - {subformType == "json"
            ? "JSON Column"
            : subformType == "embed"
              ? "Embed Document"
              : "Relationship"}
        </h3>
        <div class="form-types">
          {#if subformType == "json"}
            <div class="form-type">
              JSON Columns
              {#if outerFormJson}
                {#each $outerFormJson as field}
                  <div
                    class="field"
                    on:click={() => {
                      _outerField = field;
                      _subFormType = "json";
                    }}
                  >
                    <i class="ri-braces-line" />
                    {field}
                  </div>
                {/each}
              {/if}
            </div>
          {:else if subformType == "embed"}
            <div class="form-type">
              Embed Document In
              {#if outerFormEmbed}
                {#each $outerFormEmbed as field}
                  <div
                    class="field"
                    on:click={() => {
                      _outerField = field;
                      _subFormType = "embed";
                    }}
                  >
                    {#if outerField == field}
                      <i class="ri-check-line" />
                    {:else}
                      <i class="ri-braces-line" />
                    {/if}

                    {field}
                    {#if outerField == field}
                      - Select Schema to Embed
                    {/if}
                  </div>
                {/each}
              {/if}
            </div>
          {:else}
            <div
              class="form-type"
              class:invalid={$dataSourceStore.type != "link"}
            >
              {#if $dataSourceStore.type != "link"}
                Invalid Schema Selected
              {:else}
                Pick From Schema
              {/if}
            </div>
          {/if}
        </div>
      {/if}
    </div>
    <slot />
  {/if}
</Block>

<style>
  .super-form {
    flex: auto;
    display: flex;
    flex-direction: column;
    align-items: stretch;
    border: 1px solid var(--spectrum-global-color-gray-300);

    & > .form-header {
      display: flex;
      justify-content: space-between;
      align-items: center;
      border-bottom: 1px solid var(--spectrum-global-color-gray-300);
      background-color: var(--spectrum-global-color-gray-100);
      padding-left: 0.75rem;
      padding-right: 0.5rem;
      min-height: 2.4rem;
      & > .form-title {
        flex: auto;
        display: flex;
        flex-direction: column;

        & > .title {
          font-size: 16px;
          display: flex;
          align-items: center;
          line-height: 2rem;
          gap: 0.5rem;
          text-transform: uppercase;
          letter-spacing: 1.2px;
          color: var(--spectrum-global-color-gray-800);

          &.small {
            font-size: 15px;
            line-height: 1rem;
          }
        }

        & > .sub-title {
          font-size: 12px;
          color: var(--spectrum-global-color-gray-600);
        }
      }

      & > .form-buttons {
        display: flex;
        gap: 0.25rem;
      }
    }

    & > .form-body {
      flex: auto;
      display: flex;
      flex-direction: column;
      gap: 1rem;
      padding: 0.75rem;
    }
    &.form-field {
      border: unset;
      &.quiet {
        & > .form-header {
          display: none;
        }
      }

      & > .form-header {
        min-height: unset;
        margin: unset;
        padding: unset;
        border-bottom: unset;
        background-color: unset;

        &.multiRow {
          border-bottom: unset;
        }
        & > .form-title {
          flex: auto;
          & > .title {
            font-size: 11px;
            letter-spacing: 1px;
          }
        }
      }

      & > .form-body {
        padding: unset;
        border: unset;
      }
    }

    &.quiet {
      border: unset;
      gap: 0.5rem;
      & > .form-header {
        background-color: unset;
        border: unset;
        padding: unset;
      }

      & > .form-body {
        padding: unset;
        border: unset;
      }
    }

    &.multiRow {
      gap: 0.5rem;
    }

    &.multiStep {
      gap: 0.5rem;
    }
  }

  .form-row {
    display: flex;
    flex-direction: column;
  }

  .form-grid {
    display: grid;
    grid-template-columns: repeat(var(--form-columns), 1fr);
    column-gap: 0.75rem;
    row-gap: 2px;
    justify-items: stretch;

    &.labels-left {
      row-gap: 0.5rem;
    }
  }

  :global(.form-grid > .component > *) {
    grid-column: span 6;
  }

  .step-desc {
    color: var(--spectrum-global-color-gray-800);
    padding: 1rem;
    background-color: var(--spectrum-global-color-gray-100);
    margin-bottom: 8px;
    display: flex;
    align-items: center;
    font-size: 12px;
  }
  .empty {
    flex: auto;
    display: flex;
    flex-direction: column;
    align-items: stretch;
    gap: 0.25rem;
    padding: 1rem;
    border: 1px dashed var(--spectrum-global-color-gray-400);

    & > h2,
    h3 {
      align-self: center;
      margin-top: unset;
    }

    & > .message {
      display: flex;
      align-items: center;
      gap: 0.5rem;
      justify-content: center;
    }

    & > .form-types {
      padding: 0.5rem;
      display: grid;
      grid-template-columns: repeat(1, 1fr);
      align-items: stretch;
      gap: 1rem;

      & > .form-type {
        flex: auto;
        display: flex;
        flex-direction: column;
        border: 1px solid var(--spectrum-global-color-gray-300);
        background-color: var(--spectrum-global-color-gray-100);
        border-radius: 4px;
        gap: 0.5rem;
        padding: 0.5rem;

        &.invalid {
          color: var(--spectrum-global-color-red-400);
          border-color: var(--spectrum-global-color-red-500);
        }

        & > .field {
          display: flex;
          align-items: center;
          gap: 0.5rem;
          padding-left: 0.5rem;
          line-height: 2rem;
          border: 1px solid var(--spectrum-global-color-gray-300);
          background-color: var(--spectrum-global-color-gray-50);

          &:hover {
            background-color: var(--spectrum-global-color-blue-100);
            border: 1px solid var(--spectrum-global-color-blue-500);
            cursor: pointer;
          }
        }
      }
    }
  }
</style>
