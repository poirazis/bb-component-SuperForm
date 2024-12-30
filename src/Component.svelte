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
  export let entitySingular;
  export let entityPlural;
  export let nested;
  export let subformType = true;
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

  // Internal Properties
  let _outerField;
  let _outerLinkField;
  let _subFormType;

  let _multiRow = multiRow;
  let mounted;

  let jsonFields = memo([]);
  let embedFields = memo([]);

  $: jsonFields.set(getJSONFields(schema));
  $: embedFields.set(getEmbedFields(schema));

  let outerFormField;
  let outerFieldState;
  let outerFieldApi;
  let outerFieldSchema;
  let outerFieldValue;

  let unsubscribe;

  let loaded;
  let empty;
  let definition;
  let schema;
  let primaryDisplay;
  let rows = [{}];
  let existingRows = [];
  let titleHover;
  let viewToEdit;
  let activeTab = 0;

  let newRowForm = [];
  let existingRowForm = [];

  const comp_id = $component.id;
  const dataSourceStore = memo(dataSource);
  const formSettings = memo({});

  $: formSettings.set({
    $dataSourceStore,
    actionType,
    multiStep,
  });

  $: _actionType = $outerFormSettings?.actionType ?? actionType;

  // Maintain Step while in Builder
  $: inBuilder = $builderStore.inBuilder;
  $: isParent = superFormApi?.subforms?.length;
  $: activeTab = $builderStore.metadata?.step || 0;
  $: if (inBuilder && multiStep) {
    existingRowForm.forEach((x) => x?.formApi?.setStep(activeTab + 1));
    newRowForm.forEach((x) => x?.formApi?.setStep(activeTab + 1));
  }

  $: missingId = outerLinkField
    ? false
    : !nested && !inBuilder && _actionType != "create" && !rowId && !multiRow;

  // Initialize DataSourceStore
  $: dataSourceStore.set(dataSource);

  // Keep schema uptodate with Form role and field changes
  $: superFormApi?.fetchSchema(
    $dataSourceStore,
    outerField,
    outerLinkField,
    multiStep
  );

  $: richSteps = superFormApi.enrichSteps(steps, schema, multiStep);
  $: empty = richSteps[0]?.fields?.length < 1 && $component.children < 1;
  $: dirty = rows.filter((row) => row._isDirty).length > 0;

  // Data Loading Management
  $: query = QueryUtils.buildQuery(filter);
  $: fetch = superFormApi.createFetch(
    $dataSourceStore,
    rowId,
    _actionType,
    outerLinkField,
    multiRow
  );

  // Load fetched
  $: superFormApi.getInitialData($fetch, _actionType, data);

  $: batchMode =
    actionsMode == "batch" && _actionType != "view" && !outerLinkField;
  $: eventMode = actionsMode == "individual";

  $: superFormApi.init(multiRow, minRows, rowId, _actionType);

  $: canAddRows =
    (_actionType == "update" && canAdd && multiRow) ||
    _actionType == "create" ||
    outerLinkField;

  $: enrichedFormTitle = superFormApi.enrichFormTitle(
    formTitle,
    beautifyLabels,
    _actionType,
    $dataSourceStore,
    nested,
    outerField,
    subformType,
    existingRows,
    loaded
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
      if (!outerField) existingRows = [];
      rows =
        _actionType == "create"
          ? multiRow
            ? new Array(minRows || 1).fill({})
            : [{}]
          : [];
    },
    fetchSchema: async (dataSource) => {
      schema = undefined;
      loaded = false;

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
      loaded = true;
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
              existingRows[0] = {
                ...outerFieldValue,
                type: "row",
                tableId: dataSource.tableId,
              };
            }
          }
        } else if (outerFieldSchema?.type == "link") {
          _multiRow =
            outerFieldSchema.relationshipType == "many-to-many" &&
            multiRow !== false;
        }
        // Keep action Type in synch with Parent Form
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
      if (
        _actionType != "create" &&
        (rowId || inBuilder || multiRow || outerLinkField)
      )
        if (rowId)
          return fetchData({
            API,
            datasource,
            options: {
              query: {
                equal: {
                  _id: rowId,
                },
              },
            },
          });
        else
          return fetchData({
            API,
            datasource,
            options: {
              query,
              limit: outerLinkField ? 1000 : multiRow ? limit : 1,
            },
          });
      else return memo({ loading: true });
    },
    getInitialData: (res, type, data) => {
      if (outerField) return;
      if (res?.loading) return;

      if (type == "create") {
        existingRows = [];
        return;
      }

      if (multiRow && data) {
        existingRows = safeParse(data);
      } else if (res?.loaded) {
        existingRows = res.rows;
      }

      if (outerLinkField && res?.rows.length < 1) rows = [{}];
    },
    enrichFormTitle: (titleTemplate) => {
      let activeRow = existingRows[0];
      let context = {
        ...$allContext,
        [comp_id]: { ...activeRow },
      };

      let actionName = _actionType[0].toUpperCase() + _actionType.substr(1);

      if (nested)
        return titleTemplate
          ? processStringSync(titleTemplate, context)
          : beautifyLabels
            ? beautifyLabel(outerField || $dataSourceStore.fieldName)
            : outerField || $dataSourceStore.fieldName;

      return titleTemplate
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
    },
    saveRow: async () => {
      if (!visible) return;
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

          console.log(newRow);

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

  // Act as Form Field
  $: visible =
    $outerFormSettings?.multiStep && outerFormStep
      ? $outerFormStep == inStep
      : true;

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
    builderStore.actions.updateProp("subformType", "link");
    builderStore.actions.updateProp("_instanceName", "Subform");
  }

  $: if (
    inBuilder &&
    $component.selected &&
    !outerFormSettings &&
    nested &&
    mounted
  ) {
    builderStore.actions.updateProp("nested", false);
    builderStore.actions.updateProp("subformType", "link");
    builderStore.actions.updateProp("_instanceName", "New Super Form");
  }

  $: if (inBuilder && $component.selected && nested && _outerField) {
    builderStore.actions.updateProp("subformType", _subFormType);
    builderStore.actions.updateProp("outerField", _outerField);
    _outerField = undefined;
  }

  $: if (inBuilder && $component.selected && nested && _outerLinkField) {
    builderStore.actions.updateProp("outerLinkField", _outerLinkField);
    _outerLinkField = undefined;
  }

  $: if (inBuilder && $component.selected && nested && _multiRow) {
    builderStore.actions.updateProp("multiRow", true);
    _multiRow = undefined;
  }

  $: if (nested)
    $component.styles = {
      ...$component.styles,
      normal: {
        ...$component.styles.normal,
        "grid-column": "span " + colSpan * 6,
        "grid-row": "span " + rowSpan,
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

  onMount(() => {
    if (outerFormApi) {
      outerFormApi.registerSubform(superFormApi);
    }
    mounted = true;
  });

  onDestroy(() => {
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
      ? { ...existingRows[0], rows, existingRows }
      : {
          ...rows[0],
          rows,
          __currentStep: activeTab,
          __dirty: dirty,
        }}
  />
  {#if !missingId && visible && loaded && !empty}
    {#if richSteps[0]?.fields}
      <div class="super-form" class:form-field={outerForm} class:quiet>
        {#if enrichedFormTitle?.trim?.()}
          <div class="form-header" class:multiRow class:multiStep class:quiet>
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
                {enrichedFormTitle}
                {#if canEdit && titleHover && _actionType == "view"}
                  <i class={"ri-pencil-fill"} style="font-size: 16px;" />
                {/if}
                {#if viewToEdit}
                  <i class={"ri-arrow-go-back-fill"} style="font-size: 16px;" />
                {/if}
              </span>
              {#if formSubtitle}
                <span class="sub-title">{formSubtitle}</span>
              {/if}
            </div>
            <div class="form-buttons">
              {#if extendActions && formButtons.length}
                {#each formButtons as { type, text, icon, onClick }}
                  <SuperButton
                    {text}
                    {type}
                    {icon}
                    on:click={handleCustomAction(onClick)}
                  />
                {/each}
              {/if}

              {#if !nested && $dataSourceStore?.tableId}
                {#if !hideDefaultActions && _actionType != "view"}
                  {#if multiStep && _actionType == "create"}
                    {#if richSteps[activeTab]?.buttons}
                      {#each richSteps[activeTab]?.buttons as button}
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
                        disabled={!dirty}
                        on:click={handleSave}
                      />
                    {/if}
                  {/if}
                {/if}
              {/if}
            </div>
          </div>
        {/if}

        <div class="form-body" class:multiRow class:multiStep class:quiet>
          {#each existingRows as row, index}
            {@const deleted = batchMode && row["_deleted"]}
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
                    disabled: deleted || disabled,
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
                    {comp_id}
                    {row}
                    {disabled}
                    {multiRow}
                    {richSteps}
                    {actionsMode}
                    {rowActionButtons}
                    {compact}
                    inView={_actionType == "view"}
                    canAdd={index == existingRows.length - 1 &&
                      rows.length < 1 &&
                      canAdd}
                    {batchMode}
                    {canDelete}
                    on:delete={(e) => handleDelete(index)}
                    on:save={(e) => handleSave(index)}
                    on:add-row={handleAddRow}
                    on:valuesChanged={(e) =>
                      handleValuesChanges(index, e.detail)}
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

          {#if canAddRows}
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
                    bind:row
                    {compact}
                    {disabled}
                    canDelete={index > 0 ||
                      rows.length > 1 ||
                      existingRows.length}
                    canAdd={index == rows.length - 1}
                    {rowActionButtons}
                    {multiRow}
                    {richSteps}
                    actionsMode={nested ? "batch" : actionsMode}
                    on:step-change={(e) => (activeTab = e.detail - 1)}
                    on:delete={(e) => handleDelete(index, "new")}
                    on:save={(e) => handleSave(index, "new")}
                    on:add-row={handleAddRow}
                    on:valuesChanged={(e) => {
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
                        {#if richSteps[activeTab]?.desc}
                          <div class="step-desc">
                            {richSteps[activeTab].desc}
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
        <h2>Welcome to the Super Form</h2>
        <div class="message">
          <i class="ri-information-line" />
          Please Select or Add some Fields to get Started
        </div>
      {:else}
        <h3>
          Subform Type {subformType == "json"
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
            <div class="form-type">Pick From Schema</div>
          {/if}
        </div>
      {/if}
    </div>
    <slot />
  {/if}
</Block>

<style>
  .super-form {
    display: flex;
    flex-direction: column;
    align-items: stretch;

    & > .form-header {
      display: flex;
      justify-content: space-between;
      align-items: center;
      min-height: 2rem;
      padding-bottom: 1rem;
      margin-bottom: 0.5rem;
      border-bottom: 1px solid var(--spectrum-global-color-gray-400);

      &.multiRow {
        border-bottom: unset;
      }

      &.multiStep {
        border-bottom: unset;
        padding-bottom: unset;
        margin-bottom: unset;
      }
      & > .form-title {
        flex: auto;
        display: flex;
        flex-direction: column;

        & > .title {
          font-size: 16px;

          &.small {
            font-size: 14px;
          }
        }

        & > .sub-title {
          font-size: 12px;
          line-height: 1rem;
          color: var(--spectrum-global-color-gray-700);
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
      padding-top: 0.5rem;
    }
    &.form-field {
      margin-top: 0.85rem;
      &.quiet {
        & > .form-header {
          border: unset;
          padding-bottom: 0.25rem;
        }
      }

      & > .form-header {
        min-height: unset;
        border-bottom: 1px solid var(--spectrum-global-color-gray-300);
        padding-bottom: 0.5rem;

        &.multiRow {
          border-bottom: unset;
        }
        & > .form-title {
          flex: auto;
          font-size: 13px;
          color: var(--spectrum-global-color-gray-700);
        }
      }
    }

    &.quiet {
      & > .form-header {
        border: unset;
        padding-bottom: unset;
      }
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
    gap: 0.5rem;
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
      gap: 1rem;
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
