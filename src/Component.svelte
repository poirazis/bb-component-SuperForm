<script>
  import { getContext, onMount, onDestroy, setContext, tick } from "svelte";
  import RowFormWrapper from "./RowFormWrapper.svelte";
  import SuperFormHeader from "./SuperFormHeader.svelte";
  import NewFormWizard from "./NewFormWizard.svelte";
  import { fly } from "svelte/transition";
  import { get, derived } from "svelte/store";

  import { SuperForm } from "@poirazis/supercomponents-shared";

  const FieldTypeToComponentMap = {
    super_components: {
      string: "plugin/bb-component-SuperFieldText",
      icon: "plugin/bb-component-SuperFieldIcon",
      color: "plugin/bb-component-SuperFieldColor",
      number: "plugin/bb-component-SuperFieldNumber",
      bigint: "plugin/bb-component-SuperFieldNumber",
      options: "plugin/bb-component-SuperFieldOption",
      array: "plugin/bb-component-SuperFieldOptions",
      jsonarray: "plugin/bb-component-SuperFieldArray",
      boolean: "plugin/bb-component-SuperFieldBoolean",
      longform: "longformfield",
      datetime: "plugin/bb-component-SuperFieldDatetime",
      link: "plugin/bb-component-SuperFieldRelationship",
      self_link: "plugin/bb-component-SuperFieldRelationship",
      json: "plugin/bb-component-SuperFieldJSON",
      barcodeqr: "codescanner",
      bb_reference_single: "plugin/bb-component-SuperFieldRelationship",
      bb_reference: "plugin/bb-component-SuperFieldRelationship",
      signature_single: "signaturesinglefield",
      attachment_single: "plugin/bb-component-SuperFieldAttachment",
      attachment: "plugin/bb-component-SuperFieldAttachmentList",
      tags: "plugin/bb-component-SuperFieldTags",
    },
    budibase_components: {
      string: "stringfield",
      number: "numberfield",
      bigint: "bigintfield",
      options: "optionsfield",
      array: "multifieldselect",
      jsonarray: "jsonarray",
      boolean: "booleanfield",
      longform: "longformfield",
      datetime: "datetimefield",
      link: "relationshipfield",
      self_link: "relationship",
      json: "jsonfield",
      barcodeqr: "codescannerfield",
      bb_reference_single: "bbreferencesinglefield",
      bb_reference: "bbreferencefield",
      signature_single: "signaturesinglefield",
      attachment_single: "attachmentsinglefield",
      attachment: "attachmentfield",
    },
  };

  const specialColumns = [
    "created_at",
    "created_by",
    "updated_at",
    "updated_by",
    "deleted_at",
    "deleted_by",
  ];

  const {
    Block,
    BlockComponent,
    memo,
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

  export let showHeader = true;
  export let showFooter = true;
  export let formFooterText;
  export let formTitle;
  export let titleIcon;
  export let formSubtitle;
  export let nested;
  export let subformType = "link";
  export let extendActions;
  export let formButtons = [];
  export let buttonPosition = "top";
  export let hideDefaultActions = false;
  export let actionType = "create";
  export let dataSource;
  export let useAutocolumns = true;

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
  export let rowData;

  // Multi Step Settings
  export let multiStep;
  export let steps = [];

  // Sub Form Settings
  export let inStep = 1;
  export let colSpan = 6;
  export let rowSpan = 1;
  export let outerField;
  export let outerLinkField;

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
  export let slotSpan = 1;

  export let entitySingular;
  export let entityPlural;
  export let flex = "none";
  export let component_type = "super_components";

  // Internal Properties
  let _outerField;
  let _outerLinkField;
  let _subFormType;
  let _multiRow = multiRow;

  let mounted;
  let dirty;

  let jsonFields = memo([]);
  let embedFields = memo([]);
  let currentUser = $allContext?.user?.email || "Unknown User";

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
  let viewToEdit = false;
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
  $: superFormApi.resetState(rowId);
  $: superFormApi.switchState(viewToEdit);

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
  $: richSteps = superFormApi.enrichSteps(
    steps,
    schema,
    multiStep,
    useAutocolumns
  );

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
  $: query = outerLinkField
    ? {}
    : multiRow || (!multiRow && inBuilder && !rowId)
      ? QueryUtils.buildQuery(filter)
      : rowId
        ? QueryUtils.buildQuery(singleFilter)
        : null;

  $: fetch =
    query !== null ? superFormApi.createFetch($dataSourceStore, query) : null;
  $: superFormApi.getInitialData($fetch, multiRow, _actionType);

  // Reset state when no query (rowId unset outside builder)
  $: if (query === null) {
    loaded = _actionType === "create";
    emptyData = _actionType !== "create";
    existingRows = [];
  }

  // Check if special columns exist in the schema
  $: hasSpecialColumns =
    schema &&
    Object.keys(schema).some((field) => specialColumns.includes(field));

  // Format dates for footer
  $: formattedUpdated = existingRows[0]?.updated_at
    ? (() => {
        const date = new Date(existingRows[0].updated_at);
        return `${date.getDate().toString().padStart(2, "0")}/${(
          date.getMonth() + 1
        )
          .toString()
          .padStart(2, "0")}/${date.getFullYear().toString().slice(-2)} ${date
          .getHours()
          .toString()
          .padStart(2, "0")}:${date.getMinutes().toString().padStart(2, "0")}`;
      })()
    : null;
  $: formattedCreated = existingRows[0]?.created_at
    ? (() => {
        const date = new Date(existingRows[0].created_at);
        return `${date.getDate().toString().padStart(2, "0")}/${(
          date.getMonth() + 1
        )
          .toString()
          .padStart(2, "0")}/${date.getFullYear().toString().slice(-2)} ${date
          .getHours()
          .toString()
          .padStart(2, "0")}:${date.getMinutes().toString().padStart(2, "0")}`;
      })()
    : null;

  $: empty =
    loaded && richSteps[0]?.fields?.length < 1 && $component.children < 1;

  $: dirty = derived(
    [...newRowForm, ...existingRowForm].map((form) => form?.formState),
    (formStates) => formStates.some((state) => state?.dirty)
  );

  // If we are a nested form of a multi step form and designated to appear on a specific step.
  $: visible =
    $outerFormSettings?.multiStep && outerFormStep
      ? $outerFormStep == inStep
      : true;

  // Data Loading Management
  $: batchMode =
    actionsMode == "batch" && _actionType != "view" && !outerLinkField;
  $: eventMode = actionsMode == "individual";

  $: enrichedFormTitle = superFormApi.enrichFormTitle(
    formTitle,
    formSubtitle,
    $formSettings,
    loaded,
    emptyData,
    $fetch,
    primaryDisplay
  );

  /** The Super Form API
   *
   *
   */
  const superFormApi = {
    id: comp_id,
    get __dirty() {
      return $dirty;
    },
    init: () => {
      existingRows = [];
      existingRowForm = [];
      rows =
        _actionType == "create"
          ? multiRow
            ? new Array(minRows || 1).fill({})
            : [{}]
          : [];
    },
    updateField: async ({ type, field, value }) => {
      await tick();

      var formApi =
        _actionType == "create"
          ? newRowForm[0].formApi
          : existingRowForm[0].formApi;

      if (formApi)
        if (type === "set") {
          formApi.setFieldValue(field, value);
        } else {
          formApi.resetField(field);
        }
    },
    resetState: () => {
      if (nested) return;

      _actionType = actionType;
      viewToEdit = false;
    },
    switchState: (state) => {
      if (nested) return;

      if (state) {
        _actionType = "update";
        $formSettings.actionType = "update";
      } else {
        _actionType = actionType;
        $formSettings.actionType = actionType;
      }
      viewToEdit = state;
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
          try {
            schema = await fetchDatasourceSchema(dataSource);
          } catch (error) {
            schema = null;
          }
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
            (outerFieldSchema.relationshipType == "many-to-many" ||
              outerFieldSchema.relationshipType == "many-to-one") &&
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
          quiet: true,
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
          quiet: true,
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
                providerId: comp_id,
                tableId: $dataSourceStore.tableId,
              },
            },
          ],
        },
      ];

      if (buttons) return buttons;
      else if (total == index + 1 || total == 1)
        return total > 1 ? defaultButtonsLast : [defaultButtonsLast[1]];
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
              (!field.autocolumn ||
                (useAutocolumns && specialColumns.includes(field.name))) &&
              field.visible != false &&
              !field.nestedJSON
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
                  if (field.schema[sub_field].type != "json") {
                    if (field.schema[sub_field].type == "array" && !nested) {
                      return;
                    } else {
                      defaultFields.push({
                        label: outerField
                          ? sub_field
                          : field.name + "." + sub_field,
                        field: field.name + "." + sub_field,
                        active: true,
                      });
                    }
                  }
                });
              }
            } else {
              defaultFields.push({
                field: field.name,
                label: field.name.startsWith("fk_self_")
                  ? "Parent"
                  : field?.name || field.field,
                active: true,
              });
            }
          });
      }

      let allFields = [...fields, ...defaultFields];
      let nonSpecialFields = allFields.filter(
        (f) => !specialColumns.includes(f.field)
      );
      let specialFields = allFields.filter((f) =>
        specialColumns.includes(f.field)
      );

      return [...nonSpecialFields, ...specialFields]
        .filter((field) => field.active)
        .map((field) => {
          if (field.field.startsWith("fk_self_")) field.label = "Parent";

          const fieldName = field.field;
          const isSpecial = specialColumns.includes(fieldName);
          let extraProps = {};

          if (isSpecial && useAutocolumns) {
            extraProps.invisible = true;
            if (_actionType === "create") {
              if (fieldName === "created_by") {
                extraProps.defaultValue = currentUser;
              }
              if (fieldName === "created_at") {
                extraProps.defaultValue = new Date();
              }
            }
            if (_actionType === "update" || _actionType === "view") {
              if (fieldName === "updated_by") {
                extraProps.defaultValue = currentUser;
              }
              if (fieldName === "updated_at") {
                extraProps.defaultValue = new Date();
              }
            }
          }

          return {
            ...field,
            label:
              beautifyLabels && !labelTemplate
                ? beautifyLabel(field.label)
                : field.label,
            extraProps,
          };
        });
    },
    createFetch: (datasource, query) => {
      return fetchData({
        API,
        datasource,
        options: {
          limit: outerLinkField ? 1000 : multiRow ? limit : 1,
          query,
        },
      });
    },
    getInitialData: (res) => {
      if (!res?.loaded) return;

      if (outerField || _actionType == "create") {
        loaded = true;
        emptyData = false;
        return;
      }

      if (multiRow && rowData) {
        existingRows = safeParse(rowData);
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
            : "New " +
              (multiRow
                ? entityPlural || beautifyLabel($dataSourceStore.label)
                : entitySingular || beautifyLabel($dataSourceStore.label));

        richFormSubtitle = formSubtitle
          ? processStringSync(formSubtitle, $allContext)
          : undefined;
      }

      return title;
    },
    saveRow: async () => {
      if (!superFormApi.validate() || _actionType == "view") return;

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

      // We are managing a JSON field
      if (outerField && visible) {
        let formState = existingRowForm[0]?.formState;
        if (_actionType == "create") formState = newRowForm[0]?.formState;
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
  const getComponentForField = (field) => {
    const fieldSchemaName = field.name || field.field;
    let json_path;
    let innerType;
    let fieldSchema;

    if (!fieldSchemaName) {
      return null;
    }

    if (fieldSchemaName.includes(".")) {
      json_path = fieldSchemaName.split(".");
      fieldSchema = schema[json_path[0]]?.schema;
      innerType = fieldSchema[json_path[1]]?.type;
      if (innerType == "array") innerType = "jsonarray";
    } else {
      innerType = schema[fieldSchemaName]?.type;
      if (innerType == "array") {
        if (schema[fieldSchemaName]?.constraints?.inclusion?.length < 1)
          innerType = "jsonarray";
      }

      if (
        fieldSchemaName.toLowerCase() == "color" &&
        component_type == "super_components"
      )
        innerType = "color";
      if (
        fieldSchemaName.toLowerCase() == "icon" &&
        component_type == "super_components"
      )
        innerType = "icon";

      if (
        schema[fieldSchemaName]?.type == "string" &&
        schema[fieldSchemaName]?.subtype == "array"
      )
        innerType = "jsonarray";

      if (
        schema[fieldSchemaName]?.type == "string" &&
        schema[fieldSchemaName]?.subtype == "array" &&
        fieldSchemaName.toLowerCase() == "tags"
      )
        innerType = "tags";
    }

    const type = fieldSchemaName.startsWith("fk_self_")
      ? "self_link"
      : innerType;

    return FieldTypeToComponentMap[component_type][type];
  };

  const getPropsForField = (field, idx) => {
    return {
      ...field,
      field: field?.field,
      label: labelTemplate
        ? processStringSync(labelTemplate, { field: field?.field })
        : field?.label,
      placeholder: beautifyLabels
        ? field.label
        : (field.placeholder ?? field.label),
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
      ownId: field?.field?.includes("fk_self_")
        ? existingRows[idx]?.["id"]
        : null,
      direction: field?.direction == "vertical" ? "column" : "row",
      controlType: "select",
      ...field.extraProps,
    };
  };

  const beautifyLabel = (label) => {
    if (!beautifyLabels || !label) return label;

    let fields = label.split(".");
    fields.forEach((field, index) => {
      let words = field.split("_");
      words.forEach((word, index) => {
        if (word.length <= 3) {
          words[index] = word.toUpperCase();
        } else {
          words[index] = word[0]?.toUpperCase() + word?.slice(1);
        }
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
                providerId: comp_id,
              },
              "##eventHandlerType": "Save Row",
            },
          ];

          cmd = enrichButtonActions(event, $allContext);
          cmd_after = enrichButtonActions(afterSave, $allContext);

          await cmd?.();

          cmd_after?.();
          superFormApi?.reset();
        }
      } else {
        await superFormApi.saveRow();

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
              providerId: comp_id,
            },
            "##eventHandlerType": "Save Row",
          },
        ];

        // Expose the internal Form to the outer context
        cmd = enrichButtonActions(event, $allContext);
        cmd_after = enrichButtonActions(afterSave, $allContext);
        await cmd?.();
        await cmd_after?.();

        if (viewToEdit) {
          viewToEdit = false;
          _actionType = "view";
          $formSettings.actionType = "view";
        }

        fetch?.refresh();
      }
    }
  };

  const handleSaveMany = async () => {
    if (superFormApi.validate()) {
      if (_actionType == "create") {
        for (let i = 0; i < rows.length; i++) {
          const res = await API.saveRow({
            ...get(newRowForm[i].formState).values,
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
          const res = await API.saveRow({
            ...get(existingRowForm[i].formState).values,
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

  const handleBuilderUpdates = () => {
    if (!inBuilder || !$component.selected) return;

    if (outerFormSettings && !nested) {
      builderStore.actions.updateProp("nested", true);
    } else if (!outerFormSettings && nested) {
      builderStore.actions.updateProp("nested", false);
      builderStore.actions.updateProp("_instanceName", "New Super Form Pro");
    }

    if (nested && _outerField) {
      builderStore.actions.updateProp("subformType", _subFormType);
      builderStore.actions.updateProp("outerField", _outerField);
      builderStore.actions.updateProp(
        "_instanceName",
        `Subform JSON ${subformType === "embed" ? "Embed - " : "Column - "}${_outerField}`
      );
      _outerField = undefined;
    }

    if (nested && _outerLinkField) {
      builderStore.actions.updateProp("outerLinkField", _outerLinkField);
      builderStore.actions.updateProp(
        "_instanceName",
        `Subform Link - ${_outerLinkField}`
      );
      _outerLinkField = undefined;
    }

    if (nested && _multiRow) {
      builderStore.actions.updateProp("multiRow", true);
      _multiRow = undefined;
    }
  };

  $: handleBuilderUpdates($component?.selected);

  $: $component.styles = {
    ...$component.styles,
    normal: {
      "grid-column": nested ? "span " + colSpan * 6 : undefined,
      "grid-row": nested ? "span " + rowSpan : undefined,
      "align-items": "stretch",
      display: visible ? "flex" : "none",
      flex,
      ...$component.styles.normal,
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

  const handleFieldSelect = ({ detail }) => {
    const { field, subformType } = detail;
    _outerField = field;
    _subFormType = subformType;
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
    // If we are linked to a JSON column, update the value
    if (outerField) superFormApi?.saveRow();
    outerFormApi?.unregisterSubform(superFormApi);
  });
</script>

<!-- svelte-ignore a11y-no-static-element-interactions -->
<!-- svelte-ignore a11y-click-events-have-key-events -->
<!-- svelte-ignore a11y-mouse-events-have-key-events -->

<Block>
  {#if loaded && !empty && !emptyData}
    {#if richSteps[0]?.fields}
      <div class="super-form" class:form-field={outerForm}>
        {#if showHeader}
          <SuperFormHeader
            title={enrichedFormTitle}
            subtitle={richFormSubtitle}
            {titleIcon}
            isDirty={$dirty}
            {formButtons}
            {hideDefaultActions}
            {disabled}
            {multiStep}
            actionType={_actionType}
            {multiRow}
            {actionsMode}
            {canDelete}
            {canEdit}
            {viewToEdit}
            hasDataSource={!nested && $dataSourceStore?.tableId}
            {richSteps}
            {activeStep}
            {isParent}
            {buttonPosition}
            on:save={handleSave}
            on:saveMany={handleSaveMany}
            on:delete={() => handleDelete(0)}
            on:reset={superFormApi.reset}
            on:toggleEdit={() => {
              viewToEdit = !viewToEdit;
            }}
          />
        {/if}
        <div class="form-body">
          <!-- Existing Rows -->
          {#each existingRows as row, index}
            <div class="form-row">
              <SuperForm
                bind:form={existingRowForm[index]}
                provideContext={!multiRow}
                {row}
                actionType="Update"
                dataSource={outerField && subformType == "json"
                  ? {
                      type: "table",
                      tableId: outerForm?.dataSource?.tableId,
                    }
                  : { ...$dataSourceStore, type: "table" }}
                disabled={row["_deleted"] || disabled}
                readonly={!disabled && _actionType === "view"}
                style="width: 100%;"
              >
                <RowFormWrapper
                  {row}
                  {comp_id}
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
              </SuperForm>
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
                <SuperForm
                  bind:form={newRowForm[index]}
                  actionType="Create"
                  provideContext={!multiRow}
                  dataSource={$dataSourceStore}
                  {disabled}
                  style="width: 100%;"
                >
                  <RowFormWrapper
                    isNew
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
                    on:changeStep={(e) => {
                      if (activeStep !== e.detail) activeStep = e.detail;
                    }}
                    on:delete={(e) => handleDelete(index, "new")}
                    on:save={(e) => handleSave(index, "new")}
                    on:add-row={handleAddRow}
                    on:reset={() => {
                      newRowForm[index]?.formApi?.reset();
                      rows[index] = {};
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
                          {#each step.fields as field, idx}
                            {#if getComponentForField(field) && field.active}
                              <BlockComponent
                                type={getComponentForField(field)}
                                props={getPropsForField(field, index)}
                                name={"SuperField-" + field?.field}
                                order={idx}
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
                </SuperForm>
              </div>
            {/each}
          {/if}
        </div>
        <!-- New footer for special columns -->
        {#if showFooter}
          {#if formFooterText}
            <div class="form-footer">
              {processStringSync(formFooterText, $allContext)}
            </div>
          {:else if hasSpecialColumns && useAutocolumns && existingRows.length > 0 && (formattedUpdated || formattedCreated)}
            <div class="form-footer">
              {#if formattedUpdated}
                <i class="ph ph-pencil-simple"></i>
              {:else}
                <i class="ph ph-plus-circle"></i>
              {/if}
              {formattedUpdated || formattedCreated} <i class="ph ph-user"></i>
              {existingRows[0]?.updated_by ||
                existingRows[0]?.created_by ||
                "System"}
            </div>
          {/if}
        {/if}
      </div>
    {/if}
  {/if}

  {#if loaded && empty && inBuilder}
    <NewFormWizard
      {nested}
      {subformType}
      outerFormJson={$outerFormJson}
      outerFormEmbed={$outerFormEmbed}
      dataSource={$dataSourceStore}
      on:selectField={handleFieldSelect}
    />
    <slot />
  {/if}
</Block>

<style>
  .super-form {
    flex: auto;
    display: flex;
    flex-direction: column;
    align-items: stretch;

    --spectrum-alias-border-color: var(--spectrum-global-color-gray-300);
    overflow: hidden;
    gap: 0.75rem;

    & > .form-body {
      flex: auto;
      display: flex;
      flex-direction: column;
      gap: 1rem;
    }
    &.form-field {
      border: unset;

      & > .form-body {
        padding: unset;
        padding-top: 0.25rem;
        border: unset;
      }
    }

    &.multiRow {
      gap: 0.5rem;
    }

    &.multiStep {
      gap: 0.5rem;
    }

    &.loading {
      min-height: "360px";
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

  .form-footer {
    margin-top: 0.5rem;
    padding: 0.5rem;
    background-color: transparent;
    border-top: 1px solid var(--spectrum-global-color-gray-300);
    font-size: 12px;
    color: var(--spectrum-global-color-gray-600);
    display: flex;
    justify-content: flex-start;
    align-items: center;
    gap: 0.35rem;

    &:hover {
      color: var(--spectrum-global-color-gray-700);
    }
  }
</style>
