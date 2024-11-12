<script>
  import { getContext, createEventDispatcher } from "svelte";
  import SuperButton from "../../bb_super_components_shared/src/lib/SuperButton/SuperButton.svelte";
  const form = getContext("form");
  const { enrichButtonActions, memo } = getContext("sdk");
  const allContext = getContext("context");

  const dispatch = createEventDispatcher();

  export const formObject = form;

  export let isValid = false;
  export let isDirty = false;

  export let index;

  export let isNew;
  export let disabled;
  export let isDeleted;
  export let fieldsLength;
  export let actionsMode;

  export let canDelete;
  export let canAdd;
  export let multiRow;
  export let rowActionButtons;
  export let numbering;
  export let inView;

  export let comp_id;
  export let row;

  let valuesStore;
  let hover;
  let initialized = isNew;

  let unsubscribe = form.formState.subscribe((value) => {
    if (!valuesStore && !isEmpty(value.values)) {
      valuesStore = memo(value.values);
      return;
    }
    if (!isEmpty(value.values)) {
      valuesStore.set(value.values);
      initialized = true;
      isValid = value.valid;
    }
  });

  $: valuesChanged($valuesStore);

  const valuesChanged = (values, init) => {
    if (!isEmpty(values) && !isDeleted && initialized) {
      isDirty = true;
      dispatch("valuesChanged", { ...values });
    }
  };

  function isEmpty(obj) {
    if (obj && Object.keys(obj).length >= fieldsLength) {
      return Object.values(obj).every((x) => x === null || x === "");
    }
    return true;
  }

  const onDelete = () => {
    dispatch("delete", {});
  };

  const onSave = () => {
    dispatch("save", {});
  };

  const handleRowAction = async (event) => {
    if (!event) return;
    let context = {
      ...$allContext,
      [comp_id]: { ...row },
    };
    let cmd = enrichButtonActions(event, context);
    await cmd?.();
  };
</script>

<!-- svelte-ignore a11y-no-noninteractive-tabindex -->
<!-- svelte-ignore a11y-no-static-element-interactions -->
<!-- svelte-ignore a11y-mouse-events-have-key-events -->
<div
  class="row-wrapper"
  class:multiRow
  class:isDirty={multiRow && isDirty}
  on:mouseover={() => (hover = true)}
  on:mouseleave={() => (hover = false)}
>
  {#if multiRow}
    <div class="row-header">
      <!-- svelte-ignore a11y-click-events-have-key-events -->
      {#if rowActionButtons?.length && inView}
        {#each rowActionButtons as { onClick, ...rest }}
          <SuperButton
            {...rest}
            disabled={!onClick}
            on:click={handleRowAction(onClick)}
          />
        {/each}
      {/if}
      <!-- svelte-ignore a11y-click-events-have-key-events -->
      {#if isDeleted}
        <SuperButton icon={"ri-arrow-go-back-fill"} on:click={onDelete} quiet />
      {:else}
        {#if canDelete}
          <SuperButton
            icon={"ri-delete-bin-2-line"}
            type="warning"
            size="S"
            {disabled}
            fillOnHover
            quiet
            on:click={onDelete}
          />
        {/if}
        {#if canAdd}
          <SuperButton
            icon={"ri-add-line"}
            size="S"
            type="cta"
            {disabled}
            fillOnHover
            quiet
            on:click={() => dispatch("add-row", {})}
          />
        {/if}
      {/if}
      {#if actionsMode == "event"}
        <SuperButton
          icon={"ri-save-line"}
          on:click={onSave}
          emphasized
          quiet
          disabled={!isDirty || disabled}
        />
      {/if}
    </div>
  {/if}
  <slot />
</div>

<style>
  .row-wrapper {
    display: flex;
    flex-direction: column;
    align-items: stretch;
    color: var(--spectrum-global-color-gray-500);

    &.multiRow {
      padding: 0.85rem;
      border: 1px solid var(--spectrum-global-color-gray-300);

      &:focus-within {
        border: 1px solid var(--spectrum-global-color-gray-400);
      }
    }

    & > .row-header {
      display: flex;
      justify-content: flex-end;
    }
  }
  .isDirty {
    color: var(--spectrum-global-color-orange-700) !important;
    border-left: 2px solid var(--spectrum-global-color-orange-400) !important;
  }
</style>
