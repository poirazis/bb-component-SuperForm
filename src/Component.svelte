<script>
  import { getContext } from "svelte";
  import RowWrapper from "./RowWrapper.svelte";
  import SuperButton from "../../bb_super_components_shared/src/lib/SuperButton/SuperButton.svelte";
  import { fly } from "svelte/transition";

  const FieldTypeToComponentMap = {
    string: "plugin/bb-component-SuperFieldText",
    number: "plugin/bb-component-SuperFieldNumber",
    bigint: "plugin/bb-component-SuperFieldNumber",
    options: "plugin/bb-component-SuperFieldOption",
    array: "plugin/bb-component-SuperFieldOptions",
    jsonarray: "plugin/bb-component-SuperFieldOptions",
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

  export let formTitle;
  export let actionType = "create";
  export let dataSource;
  export let idColumn = "_id";
  export let size = "M";
  export let disabled;
  export let fields;
  export let canEdit;
  export let canDelete;
  export let showPlaceholders = true;
  export let mode = "formInput";
  export let relViewMode = "pills";
  export let optionsViewMode = "pills";
  export let showDirty = true;
  export let hideLabels = true;
  export let expandLinks = [];
  export let formColumns = 3;
  export let groupJSON;

  // Single Row Settings
  export let rowId;
  export let singleCanReset = true;
  export let singleRowActionsMode;
  export let defaultNotifications;
  export let requireConfirmation;

  // Multi Row Settings
  export let multiRow;
  export let numbering;
  export let actionsMode;
  export let rowActionButtons;
  export let minRows = 1;
  export let maxRows;
  export let canAdd = true;
  export let data;
  export let simpleRows;

  // Data Settings
  export let filter;
  export let limit = 10;

  // Events
  export let onRowSave;
  export let onRowDelete;
  export let onBatchSave;
  export let onSingleRowSave;
  export let afterSingleRowSave;
  export let onSingleRowDelete;
  export let afterSingleRowDelete;

  let loaded;
  let definition;
  let schema;
  let rows = [{}];
  let existingRows = [];
  let patchRows = [];
  let titleHover;
  let viewToEdit;

  let newRowForm = [];
  let existingRowForm = [];

  const comp_id = $component.id;
  const dataSourceStore = memo(dataSource);

  // Check Dependencies
  $: inBuilder = $builderStore.inBuilder;
  $: missingId = !inBuilder && actionType != "create" && !rowId && !multiRow;

  // Initialize DataSourceStore
  $: dataSourceStore.set(dataSource);
  $: fetchSchema($dataSourceStore);
  $: innerFields = getDefaultFields(fields, schema);

  // Data Loading Management
  $: query = QueryUtils.buildQuery(filter);
  $: fetch = createFetch($dataSourceStore);
  $: fetch?.update({
    query: !inBuilder
      ? {
          ...query,
          equal: {
            [idColumn]: rowId,
          },
        }
      : query,
    limit: multiRow ? limit : 1,
  });
  $: getInitialData($fetch, actionType, data);

  $: batchMode = actionsMode == "batch" && actionType != "view";
  $: eventMode = actionsMode == "event" || singleRowActionsMode == "event";
  $: autoSave = multiRow && actionsMode == "direct";

  $: init(multiRow, minRows, rowId);
  $: canAddRows =
    (actionType == "update" && canAdd && multiRow) || actionType == "create";

  $: gridColumns = formColumns;

  $: multiRow ? (hideLabels = true) : (hideLabels = false);
  $: enrichedFormTitle = enrichFormTitle(formTitle, actionType);

  const validateAll = () => {
    newRowForm.forEach((form) => {
      form?.formApi.validate();
    });

    existingRowForm.forEach((form) => {
      form?.formApi.validate();
    });
  };

  const resetAll = () => {
    newRowForm.forEach((form) => {
      form.formApi.reset();
    });
  };

  const actions = [
    {
      type: ActionTypes.ValidateForm,
      callback: validateAll,
    },
    { type: ActionTypes.ClearForm, callback: resetAll },
    { type: ActionTypes.UpdateFieldValue, callback: () => {} },
    { type: ActionTypes.ScrollTo, callback: () => {} },
  ];

  const init = () => {
    viewToEdit = false;
    rows =
      actionType == "create"
        ? multiRow
          ? new Array(minRows || 1).fill({})
          : [{}]
        : [];
  };

  const fetchSchema = async (dataSource) => {
    if (dataSource?.tableId) {
      try {
        definition = await API.fetchTableDefinition(dataSource.tableId);
        schema = definition?.schema;
      } catch (error) {
        definition = null;
        schema = null;
      }
    } else {
      schema = await fetchDatasourceSchema(dataSource);
    }
    if (!loaded) {
      loaded = true;
    }
  };

  const getDefaultFields = (fields, schema) => {
    if (!schema) {
      return [];
    }
    let defaultFields = [];

    if (!fields) {
      fields = [];
      Object.values(schema)
        .filter((field) => !field.autocolumn)
        .forEach((field) => {
          if (field.type == "json") {
            Object.keys(field.schema)?.forEach((sub_field) => {
              defaultFields.push({
                label: sub_field,
                field: field.name + "." + sub_field,
                active: true,
              });
            });
          } else
            defaultFields.push({
              label: field.name.endsWith("_self_")
                ? field.name.slice(0, -6)
                : field.name,

              field: field.name,
              active: true,
            });
        });
    }

    return [...fields, ...defaultFields].filter((field) => field.active);
  };

  const getComponentForField = (field) => {
    const fieldSchemaName = field.field || field.name;
    let json_path;

    if (fieldSchemaName.includes(".")) {
      json_path = fieldSchemaName.split(".");
    }

    if (!fieldSchemaName) {
      return null;
    }

    const type = fieldSchemaName.endsWith("_self_")
      ? "link"
      : json_path
        ? schema[json_path[0]].schema[json_path[1]].type
        : schema[fieldSchemaName]?.type;

    return FieldTypeToComponentMap[type];
  };

  const getPropsForField = (field, idx) => {
    let fieldProps = {
      ...field,
      field: field.field,
      label: field.label.endsWith("_self_")
        ? field.label.slice(0, -6)
        : !hideLabels || idx == 0
          ? field.label
          : undefined,
      placeholder: showPlaceholders
        ? field.label.endsWith("_self_")
          ? field.label.slice(0, -6)
          : field.placeholder || field.label
        : undefined,
      useOptionColors: true,
      disabled: actionType == "Disabled",
      readonly: field.readonly || actionType == "view",
      autocomplete: field.autocomplete,
      search: true,
      role: mode,
      relViewMode,
      optionsViewMode,
      compact: true,
      showDirty,
      ownId: field.field.includes("_self_") ? existingRows[idx]?.["_id"] : null,
      direction: field.direction == "vertical" ? "column" : "row",
      controlType: expandLinks.includes(field.field)
        ? "expanded"
        : field.optionsType || "select",
    };
    return fieldProps;
  };

  const getInitialData = (res, type, data) => {
    if (type == "create") {
      existingRows = [];
      return;
    }

    if (multiRow && (data || data == "")) {
      existingRows = safeParse(data);
    } else if (res?.loaded) {
      existingRows = res.rows;
    }
  };

  const enrichFormTitle = (titleTemplate) => {
    let activeRow = existingRows[0];
    let context = {
      ...$allContext,
      [comp_id]: { ...activeRow },
    };

    return titleTemplate
      ? processStringSync(titleTemplate, context)
      : actionType == "view"
        ? "View " + $dataSourceStore.label
        : actionType == "update"
          ? "Update " + $dataSourceStore.label
          : "New " + $dataSourceStore.label;
  };

  const handleRowChange = (index, change, type) => {
    if (type == "new") {
      rows[index] = { ...rows[index], ...change };
    } else if (batchMode) {
      let id = existingRows[index]._id;
      let pos = patchRows.findIndex((x) => x._id == id);
      if (pos > -1) {
        patchRows.splice(pos, 1);
      }

      if (existingRows[index]["_isValid"] && existingRows[index]["_isDirty"])
        patchRows.push({
          _id: existingRows[index]._id,
          ...change,
        });
      existingRows[index] = {
        ...existingRows[index],
        ...change,
      };
      patchRows = patchRows;
    } else if (autoSave) {
      console.log("Auto Save not implemented yet");
    } else {
      existingRows[index] = {
        ...existingRows[index],
        ...change,
      };
    }
  };

  const safeParse = (data) => {
    try {
      return JSON.parse(data);
    } catch (ex) {
      return [];
    }
  };
  const createFetch = (datasource) => {
    return fetchData({
      API,
      datasource,
      options: {
        query,
        limit: multiRow ? limit : 1,
      },
    });
  };

  const isWhitespaceString = (str) => !/\S/.test(str);

  // Single Row action handlers
  const handleNewRowSave = async () => {
    if (newRowForm[0].formApi.validate()) {
      let cmd;
      let cmd_after;
      let context = {
        ...$allContext,
        [comp_id]: { ...rows[0] },
      };
      if (singleRowActionsMode == "direct") {
        let event = [
          {
            parameters: {
              confirm: true,
              notificationOverride: null,
              tableId: $dataSourceStore.tableId,
              providerId: comp_id,
            },
            "##eventHandlerType": "Save Row",
          },
        ];
        cmd = enrichButtonActions(event, context);
        cmd_after = enrichButtonActions(afterSingleRowSave, context);
      } else {
        cmd = enrichButtonActions(onSingleRowSave, context);
      }
      await cmd?.();
      await cmd_after?.();
      newRowForm[0].formApi.reset();
    }
  };

  const handleExistingRowSave = async () => {
    if (existingRowForm[0].formApi.validate()) {
      let cmd;
      let cmd_after;
      let context = {
        ...$allContext,
        [comp_id]: { ...existingRows[0] },
      };

      if (singleRowActionsMode == "direct") {
        let autoOnSingleRowSave = [
          {
            parameters: {
              confirm: requireConfirmation,
              notificationOverride: !defaultNotifications,
              tableId: $dataSourceStore.tableId,
              providerId: comp_id,
            },
            "##eventHandlerType": "Save Row",
          },
        ];

        cmd = enrichButtonActions(autoOnSingleRowSave, context);
        cmd_after = enrichButtonActions(afterSingleRowSave, context);
      } else {
        cmd = enrichButtonActions(onSingleRowSave, context);
      }

      if (viewToEdit) {
        viewToEdit = false;
        actionType = "view";
      }

      await cmd?.();
      await cmd_after?.();
    }
  };

  const handleExistingRowDelete = async () => {
    let activeRow = existingRows[0];
    let cmd;
    let cmd_after;
    let context = {
      ...$allContext,
      [comp_id]: { ...activeRow },
    };

    if (singleRowActionsMode == "direct") {
      let event = [
        {
          parameters: {
            confirm: true,
            notificationOverride: null,
            tableId: "ta_7af2ccbc8f894d4abc8993855a4489c2",
            rowId: existingRows[0][idColumn],
          },
          "##eventHandlerType": "Delete Row",
        },
      ];

      (cmd = enrichButtonActions(event, context)),
        (cmd_after = enrichButtonActions(afterSingleRowDelete, context));
    } else {
      cmd = enrichButtonActions(onSingleRowDelete, context);
    }
    await cmd?.();
    await cmd_after?.();
    fetch.refresh();
  };

  // Multi Row action handlers
  const handleDelete = async (index, type) => {
    if (type == "new") {
      if (rows.length == minRows && actionType != "update") {
        notificationStore.actions.warning("Minimum " + minRows + " rows");
      } else {
        rows.splice(index, 1);
        rows = rows;
      }
    } else {
      if (batchMode) {
        existingRows[index]["_deleted"] = !existingRows[index]["_deleted"];
        pendingChanges = true;
      } else if (eventMode) {
        let context = {
          ...$allContext,
          [comp_id]: { ...existingRows[index] },
        };
        let cmd = enrichButtonActions(onRowDelete, context);
        await cmd();
        existingRows.splice(index, 1);
      } else if (autoSave) {
        let event = [
          {
            parameters: {
              confirm: true,
              notificationOverride: null,
              tableId: "ta_7af2ccbc8f894d4abc8993855a4489c2",
              rowId: existingRows[index][idColumn],
            },
            "##eventHandlerType": "Delete Row",
          },
        ];
        let cmd = enrichButtonActions(event, $allContext);
        await cmd?.();
        fetch.refresh();
      }
    }
  };

  const handleSave = async (idx, type) => {
    let activeRow = type == "new" ? rows[idx] : existingRows[idx];
    let context = {
      ...$allContext,
      [comp_id]: { ...activeRow },
    };
    let cmd = enrichButtonActions(onRowSave, context);
    await cmd();
    activeRow["_isDirty"] = false;
    existingRows = existingRows;
  };

  const handleBatchSave = async () => {
    const tableId = $dataSourceStore.tableId;
    let refresh = false;

    let validRows = rows.filter((row) => {
      return row["_isValid"] && row["_isDirty"];
    });

    let rowsForDeletion = existingRows
      .filter((x) => x["_deleted"])
      .map((x) => {
        return { _id: x["_id"] };
      });

    // Apply Deletion
    if (actionType == "update" && rowsForDeletion.length) {
      await API.deleteRows({
        tableId,
        rows: rowsForDeletion.map(({ _id }) => {
          return { _id };
        }),
      });
      refresh = true;
    }
    if (actionType != "view" && validRows.length && tableId) {
      await API.importTableData({
        tableId,
        rows: validRows,
      });
      newRowForm[0].formApi.reset();
      rows = multiRow ? [] : [{}];
      refresh = true;
    }

    // Apply Patches
    if (actionType == "update" && patchRows.length) {
      patchRows.forEach((row) => {
        API.patchRow({
          ...row,
          tableId,
        });
      });
      patchRows = [];
      refresh = true;
    }

    if ((actionType == "update" && refresh) || onBatchSave)
      setTimeout(() => {
        fetch.refresh();
        onBatchSave?.();
      }, 150);
  };
</script>

<!-- svelte-ignore a11y-no-static-element-interactions -->
<!-- svelte-ignore a11y-click-events-have-key-events -->
<!-- svelte-ignore a11y-mouse-events-have-key-events -->

<Block>
  <Provider
    data={{ rows, existingRows }}
    {actions}
    scope={ContextScopes.Global}
  />

  {#if !missingId}
    {#if innerFields.length}
      <div class="super-form">
        <div class="single-row-actions">
          <div
            class="single-row-title"
            style:cursor={canEdit ? "pointer" : null}
            on:mouseover={() => (titleHover = canEdit)}
            on:mouseleave={() => (titleHover = false)}
            on:click|self={canEdit
              ? () => {
                  actionType = viewToEdit ? "view" : "update";
                  viewToEdit = !viewToEdit;
                }
              : () => {}}
          >
            {enrichedFormTitle}
            {#if canEdit && titleHover && actionType == "view"}
              <i class={"ri-pencil-fill"} style="font-size: 16px;" />
            {/if}
            {#if viewToEdit}
              <i class={"ri-arrow-go-back-fill"} style="font-size: 16px;" />
            {/if}
          </div>
          <div>
            {#if actionType != "view" && canDelete}
              <SuperButton
                icon="ri-delete-bin-line"
                type="warning"
                fillOnHover
                quiet
                text="Delete"
                on:click={handleExistingRowDelete}
              />
            {/if}
            {#if actionType != "view"}
              <SuperButton
                icon="ri-save-line"
                text="Save"
                type="cta"
                {disabled}
                on:click={handleExistingRowSave}
              />
            {/if}
          </div>
        </div>

        {#if existingRows.length}
          {#each existingRows as row, index (index)}
            {@const deleted = batchMode && row["_deleted"]}
            <div class="form-row">
              <Provider data={{ ...row, index }} scope={ContextScopes.Local}>
                <BlockComponent
                  type="form"
                  props={{
                    actionType: "Update",
                    dataSource: $dataSourceStore,
                    size,
                    disabled: deleted || disabled,
                    readonly: !disabled && actionType === "view",
                  }}
                  styles={{
                    normal: {
                      width: "100%",
                    },
                  }}
                >
                  <RowWrapper
                    isDeleted={deleted}
                    canAdd={index == existingRows.length - 1 && rows.length < 1}
                    {disabled}
                    {batchMode}
                    {autoSave}
                    {canDelete}
                    {multiRow}
                    {numbering}
                    {actionsMode}
                    {rowActionButtons}
                    {comp_id}
                    {row}
                    inView={actionType == "view"}
                    fieldsLength={innerFields.length}
                    bind:formObject={existingRowForm[index]}
                    bind:isValid={row["_isValid"]}
                    bind:isDirty={row["_isDirty"]}
                    on:valuesChanged={(e) => handleRowChange(index, e.detail)}
                    on:delete={(e) => handleDelete(index)}
                    on:save={(e) => handleSave(index)}
                    index={index == 0 && !multiRow ? undefined : index}
                    on:add-row={() => {
                      rows.push({});
                      rows = rows;
                    }}
                  >
                    <BlockComponent
                      type="plugin/bb-component-SuperContainer"
                      name="cntFieldGroup"
                      containsSlot
                      props={{
                        flex: "grow",
                        flexFactor: 1,
                        gridColumns,
                        mode: "fieldgroup",
                        gap: "1",
                      }}
                    >
                      {#each innerFields as field, idx}
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
                      <slot />
                    </BlockComponent>
                  </RowWrapper>
                </BlockComponent>
              </Provider>
            </div>
          {/each}
        {/if}

        {#if canAddRows}
          {#each rows as row, index (index)}
            <div
              class="form-row new"
              transition:fly={{ duration: 130, y: -40 }}
            >
              <BlockComponent
                type="form"
                context={index == 0 ? "form" : undefined}
                props={{
                  actionType: "Create",
                  dataSource: $dataSourceStore,
                  size,
                  disabled,
                  readonly: !disabled && actionType === "View",
                }}
                styles={{
                  normal: {
                    width: "100%",
                  },
                }}
              >
                <RowWrapper
                  bind:formObject={newRowForm[index]}
                  bind:isValid={row["_isValid"]}
                  bind:isDirty={row["_isDirty"]}
                  isNew
                  canDelete={index > 0 ||
                    rows.length > 1 ||
                    existingRows.length}
                  canAdd={index == rows.length - 1}
                  {batchMode}
                  {autoSave}
                  {multiRow}
                  {numbering}
                  {actionsMode}
                  fieldsLength={innerFields.length}
                  on:delete={(e) => handleDelete(index, "new")}
                  on:valuesChanged={(e) => {
                    handleRowChange(index, e.detail, "new");
                    row["_isDirty"] = true;
                  }}
                  on:save={(e) => handleSave(index, "new")}
                  on:add-row={() => {
                    rows.push({});
                    rows = rows;
                  }}
                  index={multiRow ? index + existingRows?.length : undefined}
                >
                  <BlockComponent
                    type="plugin/bb-component-SuperContainer"
                    name="cntFieldGroup"
                    containsSlot
                    props={{
                      flex: "grow",
                      flexFactor: 1,
                      gridColumns,
                      mode: "fieldgroup",
                      gap: "0.5",
                    }}
                  >
                    {#each innerFields as field, idx}
                      {#if getComponentForField(field) && field.active}
                        <BlockComponent
                          type={getComponentForField(field)}
                          props={{
                            ...getPropsForField(
                              field,
                              index + existingRows.length
                            ),
                          }}
                          order={idx}
                          interactive
                          name={"SuperField-" + field?.field}
                        />
                      {/if}
                    {/each}
                    <slot />
                  </BlockComponent>
                </RowWrapper>
              </BlockComponent>
            </div>
          {/each}
        {/if}
      </div>
    {/if}

    {#if !innerFields.length && !$component.children && inBuilder}
      <div class="empty" style="padding: 0.5rem 0.85rem;">
        <h2>Welcome to the Super Form</h2>
        <div class="message">
          <i class="ri-information-line" />
          Please Select or Add some Fields to get Started
        </div>
      </div>
    {/if}
  {/if}
</Block>

<style>
  .super-form {
    display: flex;
    flex-direction: column;
    align-items: stretch;
    gap: 1rem;
  }
  .form-row {
    min-height: 2.4rem;
    display: flex;
    flex-direction: row;
    border: 1px solid transparent;
    align-items: flex-end;
    transition: all 230ms;
  }

  .single-row-actions {
    display: flex;
    flex-direction: row;
    justify-content: space-between;
    align-items: center;
  }
  .single-row-title {
    display: flex;
    flex-direction: row;
    justify-content: space-between;
    align-items: center;
    font-size: 17px;
    color: var(--spectrum-global-color-gray-600);
    font-weight: 800;
    gap: 0.35rem;
    line-height: 2rem;
  }
  .single-row-title:hover {
    color: var(--spectrum-global-color-gray-800);
  }

  .empty {
    display: flex;
    flex-direction: column;
    align-items: center;
    line-height: 2rem;
    gap: 0.5rem;

    & > .message {
      display: flex;
      align-items: center;
      gap: 1rem;
    }
  }
</style>
