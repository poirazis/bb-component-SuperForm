<script>
  import { getContext, createEventDispatcher } from "svelte";
  const form = getContext("form");
  const { memo } = getContext("sdk");

  const dispatch = createEventDispatcher();

  export let formObject = form;
  export let index;
  export let isValid = true;
  export let isDirty;
  export let isNew;
  export let isDeleted;
  export let fieldsLength;
  export let batchMode;
  export let autoSave;
  export let canDelete;

  let valuesStore;
  let initalized = isNew;
  let hover;

  let formState = form.formState;
  let unsubscribe = formState.subscribe((value) => {
    if (!valuesStore && !isEmpty(value.values)) {
      valuesStore = memo(value.values);
      return;
    }
    isValid = value.valid;
    if (!isEmpty(value.values)) {
      initalized = true;
      valuesStore.set(value.values);
    }
  });

  $: valuesChanged($valuesStore);

  const valuesChanged = (values) => {
    if (!isEmpty(values) && initalized) {
      isDirty = false;
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
</script>

<!-- svelte-ignore a11y-no-noninteractive-tabindex -->
<div class="row-wrapper" tabindex="0">
  <!-- svelte-ignore a11y-mouse-events-have-key-events -->
  <!-- svelte-ignore a11y-no-static-element-interactions -->
  {#if index > -1}
    <div
      class="row-number"
      class:isNew
      class:isValid
      on:mouseover={() => (hover = true)}
      on:mouseleave={() => (hover = false)}
    >
      {#if isDeleted}
        <i
          class="ri-arrow-go-back-fill"
          style:color={"var(--spectrum-global-color-gray-800)"}
          style:cursor={"pointer"}
          on:mousedown={onDelete}
        />
      {:else if isValid == false}
        <i
          class="ri-error-warning-fill"
          style:color={"var(--spectrum-global-color-red-400)"}
        />
      {:else if isDirty}
        <i
          class="ri-edit-fill"
          style:color={"var(--spectrum-global-color-yellow-400)"}
        />
      {:else if hover && canDelete}
        <i
          class="ri-delete-bin-line"
          style:color={"var(--spectrum-global-color-red-400)"}
          style:cursor={"pointer"}
          on:mousedown={onDelete}
        />
      {:else}
        {index + 1}
      {/if}
    </div>
  {/if}
  <slot />
  <!-- svelte-ignore a11y-no-static-element-interactions -->
  {#if !batchMode && !autoSave}
    <!-- svelte-ignore a11y-click-events-have-key-events -->
    <div class="actions" on:click={onSave}>
      <i class="ri-save-line" />
    </div>
  {/if}
</div>

<style>
  .row-wrapper {
    display: flex;
    justify-content: stretch;
    align-items: stretch;
    color: var(--spectrum-global-color-gray-500);
    gap: 0.25rem;
  }
  .row-wrapper:focus {
    outline: none;
  }
  .row-wrapper:focus-within {
    color: var(--spectrum-global-color-gray-900);
  }
  .row-number {
    min-width: 2rem;
    max-width: 2rem;
    flex: auto;
    display: flex;
    align-items: center;
    justify-content: center;
    margin: 2px;
  }

  .actions {
    line-height: 2rem;
    min-width: 2.4rem;
    display: flex;
    align-items: center;
    justify-content: center;
    border: 1px solid transparent;
  }

  .actions:hover {
    background-color: var(--spectrum-global-color-gray-50);
    border: 1px solid var(--spectrum-global-color-gray-200);
    border-radius: 4px;
    cursor: pointer;
    color: var(--primaryColor);
  }
  .isDirty {
    border-right: 2px solid var(--spectrum-global-color-orange-400) !important;
  }
  .isNew {
    background-color: var(--spectrum-global-color-blue-100);
    border-radius: 4px;
  }

  .row-wrapper:hover {
    color: var(--spectrum-global-color-gray-800);
  }
</style>
