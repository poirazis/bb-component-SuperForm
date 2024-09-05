<script>
  import { getContext } from "svelte";
  import RowWrapper from "./RowWrapper.svelte";
  import SuperButton from "../../bb_super_components_shared/src/lib/SuperButton/SuperButton.svelte";
  import { get } from "svelte/store";
  import { slide } from "svelte/transition";
  import CellString from "../../bb_super_components_shared/src/lib/SuperTableCells/CellString.svelte";

  const {
    styleable,
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
  } = getContext("sdk");
  const component = getContext("component");
  const allContext = getContext("context");

  export let rowLines = 1;
  export let minRows = 1;
  export let maxRows;
  export let canAdd = true;
  export let canDelete = true;
  export let data;
  export let actionType = "create";
  export let formType = "simple";
  export let batchMode;
  export let autoSave;
  export let dataSource;
  export let size = "M";
  export let disabled;
  export let fields = [];
  export let showPlaceholders = true;
  export let mode = "formInput";
  export let relViewMode = "pills";
  export let optionsViewMode = "pills";
  export let useOptionColors = true;
  export let showDirty = true;
  export let hideLabels = true;
  export let expandLinks = [];
  export let formColumns = 3;

  // Data Settings
  export let filter;
  export let limit = 10;
  export let rowId;

  // Events
  export let onRowSave;
  export let onBatchSave;

  let loaded;
  let definition;
  let schema;
  let rows = [{}];
  let existingRows = [];
  let patchRows = [];
  let saving;

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
    link_self: "plugin/bb-component-SuperFieldAdvancedRelationship",
    json: "jsonfield",
    barcodeqr: "codescanner",
    bb_reference_single: "plugin/bb-component-SuperFieldRelationship",
    bb_reference: "plugin/bb-component-SuperFieldRelationship",
    signature_single: "signaturesinglefield",
    attachment_single: "plugin/bb-component-SuperFieldAttachment",
    attachment: "plugin/bb-component-SuperFieldAttachmentList",
  };

  const comp_id = $component.id;
  const dataSourceStore = memo(dataSource);

  $: init(formType, actionType, data, minRows);
  $: hideNewRows =
    (formType == "simple" && actionType != "create") || actionType == "view";
  $: multiRow = formType == "multiRow";
  $: dataSourceStore.set(dataSource);
  $: fetchSchema($dataSourceStore);
  $: innerFields = getDefaultFields(fields, schema);
  $: inBuilder = $builderStore.inBuilder;
  $: gridColumns =
    formType == "multiRow"
      ? innerFields?.length + $component?.children
      : formColumns;

  $: multiRow ? (hideLabels = true) : (hideLabels = false);

  // Data Loading Management
  $: query = QueryUtils.buildQuery(filter);
  $: fetch = createFetch($dataSourceStore, actionType, formType);
  $: fetch?.update({
    query: rowId
      ? {
          ...query,
          equal: {
            _id: rowId,
          },
        }
      : query,
    limit: multiRow ? limit : 1,
  });
  $: getInitialData($fetch);

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

    if (!fields || fields.length === 0) {
      Object.values(schema)
        .filter((field) => !field.autocolumn)
        .forEach((field) => {
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
    if (!fieldSchemaName || !schema?.[fieldSchemaName]) {
      return null;
    }
    const type = fieldSchemaName.endsWith("_self_")
      ? "link_self"
      : schema[fieldSchemaName].type;

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
      labelPosition: "fieldGroup",
      search: true,
      role: mode,
      relViewMode,
      optionsViewMode,
      compact: true,
      showDirty,
      controlType: expandLinks.includes(field.field) ? "expanded" : "select",
    };
    return fieldProps;
  };

  const getInitialData = (res) => {
    if (res?.loading) return;
    existingRows = res?.rows || [];
  };

  const handleDelete = (index, type) => {
    if (type == "new") {
      rows.splice(index, 1);
    } else {
      if (batchMode) {
        existingRows[index]["_deleted"] = !existingRows[index]["_deleted"];
        existingRows[index]["_isValid"] = true;
        pendingChanges = true;
      } else {
        existingRows.splice(index, 1);
      }
      existingRows = existingRows;
    }
    rows = rows;
  };

  const handleSave = (idx, type) => {
    let activeRow = type == "new" ? rows[idx] : existingRows[idx];
    let context = {
      ...$allContext,
      [comp_id]: { ...activeRow },
    };
    let cmd = enrichButtonActions(onRowSave, context);
    cmd();
    activeRow["_isDirty"] = false;
    existingRows = existingRows;
    rows = rows;
  };

  const handleBatchSave = async () => {
    const tableId = $dataSourceStore.tableId;
    let rowsForDeletion = existingRows
      .filter((x) => x["_deleted"])
      .map((x) => {
        return { _id: x["_id"] };
      });

    if (actionType == "create") {
      if (tableId)
        await API.importTableData({
          tableId,
          rows,
        });
      rows = [{}];
    }

    // Apply Deletion
    if (actionType == "update" && rowsForDeletion.length) {
      await API.deleteRows({
        tableId,
        rows: rowsForDeletion.map(({ _id }) => {
          return { _id };
        }),
      });
    }

    // Apply Patches
    if (actionType == "update" && patchRows.length)
      patchRows.forEach(async (row) => {
        await API.patchRow({
          ...row,
          tableId,
        });
      });

    // Import New Rows
    if (actionType == "update" && rows.length) {
      await API.importTableData({
        tableId,
        rows,
      });
      rows = [];
    }

    patchRows = [];
    setTimeout(() => {
      fetch.refresh();
    }, 150);
    await onBatchSave?.();
  };

  const init = (formType, actionType, data) => {
    formType == "multiRow" && actionType == "create" && minRows
      ? (rows = new Array(minRows).fill({}))
      : formType == "simple"
        ? (rows = [{}])
        : (rows = []);

    formType == "multiRow" && actionType != "create" && data
      ? (existingRows = JSON.parse(data))
      : (existingRows = []);
  };

  const handleRowChange = (index, change, type) => {
    if (type == "new") {
      rows[index] = { ...change, _isDirty: true };
    } else if (batchMode) {
      let id = existingRows[index]._id;
      let pos = patchRows.findIndex((x) => x._id == id);
      if (pos > -1) {
        patchRows.splice(pos, 1);
      }
      patchRows.push({
        _id: existingRows[index]._id,
        ...change,
      });
      existingRows[index] = {
        ...existingRows[index],
        ...change,
        _isDirty: true,
      };
      patchRows = patchRows;
    } else if (autoSave) {
      // TODO: AutoSave
    } else {
      existingRows[index] = {
        ...existingRows[index],
        ...change,
        _isDirty: true,
      };
    }
  };

  const createFetch = (datasource, rowId) => {
    if (actionType == "create") return;

    return fetchData({
      API,
      datasource,
      options: {
        query,
        limit: multiRow ? limit : 1,
      },
    });
  };
</script>

<Block>
  {#if innerFields.length}
    <div class="super-form" use:styleable={$component.styles}>
      {#if existingRows.length}
        {#each existingRows as row, index (index)}
          {@const deleted = batchMode && row["_deleted"]}
          {@const isDirty = row["_isDirty"]}
          <div class="form-row">
            <Provider data={{ ...row, index }} scope={ContextScopes.Local}>
              <BlockComponent
                type="form"
                context={index == 0 ? "form" : undefined}
                props={{
                  actionType: actionType === "create" ? "Create" : "Update",
                  dataSource: $dataSourceStore,
                  size,
                  disabled: deleted,
                  readonly: !disabled && actionType === "View",
                }}
                styles={{
                  normal: {
                    width: "100%",
                  },
                }}
              >
                <RowWrapper
                  isDeleted={deleted}
                  {batchMode}
                  {autoSave}
                  {canDelete}
                  {isDirty}
                  fieldsLength={innerFields.length}
                  on:valuesChanged={(e) => handleRowChange(index, e.detail)}
                  on:delete={(e) => handleDelete(index)}
                  on:save={(e) => handleSave(index)}
                  index={index == 0 && formType == "simple" ? undefined : index}
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
                            ...getPropsForField(field, index),
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
            </Provider>
          </div>
        {/each}
      {/if}

      {#if !hideNewRows}
        {#each rows as row, index (index)}
          {@const isDirty = row["_isDirty"]}
          <div class="form-row new" out:slide={{ duration: 180, y: 300 }}>
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
                isNew
                canDelete
                {isDirty}
                {batchMode}
                {autoSave}
                fieldsLength={innerFields.length}
                on:delete={(e) => handleDelete(index, "new")}
                on:valuesChanged={(e) =>
                  handleRowChange(index, e.detail, "new")}
                on:save={(e) => handleSave(index, "new")}
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

      {#if formType == "multiRow" && actionType != "view" && (batchMode || canAdd)}
        <div
          class="form-row-actions"
          style:gap={"0.5rem"}
          style:margin-top={"8px"}
        >
          <SuperButton
            icon={"ri-add-fill"}
            text={"Add New"}
            fullWidth={true}
            disabled={!canAdd}
            on:click={() => {
              if (maxRows > 0 && rows.length == maxRows) {
                notificationStore.actions.warning(
                  "Maximum " + maxRows + " rows allowed"
                );
              } else {
                rows.push({});
                rows = rows;
              }
            }}
          />
          {#if batchMode}
            <SuperButton
              fullWidth={true}
              icon="ri-save-line"
              text="Save Changes"
              on:click={handleBatchSave}
            />
          {/if}
        </div>
      {/if}
    </div>
  {/if}

  {#if !innerFields.length && !$component.children}
    <div class="empty" style="padding: 0.5rem 0.85rem;">
      <i class="ri-information-line" />
      Please Select or Add some Fields to get Started
    </div>
  {/if}
</Block>

<style>
  .super-form {
    display: flex;
    flex-direction: column;
    align-items: stretch;
  }
  .form-row {
    padding-top: 2px;
    padding-bottom: 2px;
    display: flex;
    flex-direction: row;
    border: 1px solid transparent;
    align-items: flex-end;
  }

  .form-row-actions {
    display: flex;
    flex-direction: row;
    justify-content: flex-end;
  }

  .empty {
    display: flex;
    align-items: center;
    line-height: 2rem;
    gap: 0.5rem;
  }
</style>
