<script>
  import { getContext, createEventDispatcher, onMount } from "svelte";
  import SuperButton from "../../bb_super_components_shared/src/lib/SuperButton/SuperButton.svelte";

  const { enrichButtonActions, builderStore } = getContext("sdk");
  const allContext = getContext("context");

  const dispatch = createEventDispatcher();
  export let disabled;
  export let actionsMode;
  export let inView;
  export let compact;
  export let quiet;

  export let isNew;

  // Multi Step Forms
  export let richSteps;

  export let canDelete;
  export let canAdd;
  export let multiRow;
  export let rowActionButtons;
  export let rowActions;

  export let comp_id;
  export let row;

  let activeStep = 1;
  let initialized;
  $: inBuilder = $builderStore.inBuilder;
  $: isDeleted = row._deleted;
  $: multiStep = richSteps.length > 1 || richSteps[0].title;
  $: isDirty = row._isDirty;

  const formClientApi = {
    // Grab a reference to the wrapping form on initialization
    form: getContext("form"),
    init: (form) => {
      let unsubscribe = form.formState.subscribe((value) => {
        if (!initialized && !isNew) {
          initialized = true;
          return;
        }
        let values = unflattenObject(value.values);
        formClientApi.valuesChanged(values, value.valid);
        activeStep = value.currentStep;
        dispatch("step-change", activeStep);
      });
    },
    valuesChanged: (values, _isValid) => {
      let new_row = { ...values, _isDirty: !isEmpty(values), _isValid };
      dispatch("valuesChanged", new_row);
    },
  };

  const isEmpty = (obj) => {
    if (obj && Object.keys(obj).length) {
      return Object.values(obj).every((x) =>
        typeof x == "object" ? isEmpty(x) : x === null || x === ""
      );
    }
    return true;
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

  export const formObject = formClientApi.form;

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

  onMount(() => {
    formClientApi.init(formClientApi.form);
  });
</script>

<!-- svelte-ignore a11y-no-noninteractive-tabindex -->
<!-- svelte-ignore a11y-no-static-element-interactions -->
<!-- svelte-ignore a11y-mouse-events-have-key-events -->
<!-- svelte-ignore a11y-click-events-have-key-events -->
<div class="row-wrapper" class:multiRow class:isDirty class:compact>
  {#if multiRow || multiStep}
    <div class="row-header" class:multiStep class:isNew>
      {#if multiStep}
        <div
          class="form-tabs"
          class:isNew
          class:complete={activeStep == richSteps.length}
        >
          {#each richSteps as { title }, idx}
            <div
              class="tab"
              class:complete={activeStep - 1 > idx}
              class:active={idx == activeStep - 1}
              on:click={() => {
                if (inBuilder) return;
                if (formObject.formApi.validate()) {
                  formObject.formApi.setStep(idx + 1);
                }
              }}
            >
              {title}
            </div>
          {/each}
        </div>
      {/if}
      <!-- svelte-ignore a11y-click-events-have-key-events -->

      {#if multiRow}
        <span class="row-buttons">
          {#if rowActionButtons?.length}
            {#each rowActionButtons as { onClick, ...rest }}
              <SuperButton
                {...rest}
                size="S"
                on:click={handleRowAction(onClick)}
              />
            {/each}
          {/if}

          {#if rowActions}
            {#each Object.entries(rowActions) as [name, action]}
              <SuperButton
                icon="ri-flashlight-line"
                type="primary"
                quiet
                size="S"
                text={action.name}
              />
            {/each}
          {/if}

          {#if !inView}
            {#if isDeleted}
              <SuperButton
                icon={"ri-arrow-go-back-fill"}
                size="S"
                on:click={onDelete}
                quiet
              />
            {:else if canDelete}
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
            {#if actionsMode == "individual"}
              <SuperButton
                icon={"ri-save-line"}
                {disabled}
                size="S"
                type="primary"
                on:click={onSave}
                quiet
              />
            {/if}
            {#if canAdd}
              <SuperButton
                icon={"ri-add-line"}
                size="S"
                type="primary"
                disabled={!canAdd || disabled}
                fillOnHover
                quiet
                on:click={() => dispatch("add-row", {})}
              />
            {/if}
          {/if}
        </span>
      {/if}
    </div>
  {/if}

  <div class="row-body" class:multiStep class:quiet>
    <slot />
  </div>
</div>

<style>
  .row-wrapper {
    display: flex;
    flex-direction: column;
    align-items: stretch;

    &.compact {
      flex-direction: row-reverse;
      padding: unset !important;
      border: unset !important;

      & > .row-header {
        align-items: flex-end;
        padding-bottom: 4px;
      }
    }

    &.multiRow {
      padding: 0.85rem;
      padding-top: 0.5rem;
      border: 1px solid var(--spectrum-global-color-gray-200);

      &:focus-within {
        border: 1px solid var(--spectrum-global-color-gray-200);
      }

      &.isDirty {
        border: 1px solid var(--spectrum-global-color-gray-300) !important;
        background-color: var(--spectrum-global-color-gray-100);
      }
    }

    & > .row-header {
      display: flex;
      justify-content: end;

      &.multiStep {
        justify-content: space-between;
        align-items: flex-start;
        margin-top: 8px;

        &.isNew {
          justify-content: center;
          margin-bottom: 1.5rem;
          border-bottom: unset;
        }

        & > span {
          margin-top: -4px;
        }
      }

      & > .row-buttons {
        display: flex;
        gap: 0.25rem;
      }
    }

    & > .row-body {
      flex: auto;

      &.multiStep {
        flex: auto;
        padding: 1rem;
        border: 1px solid var(--spectrum-global-color-gray-300);
      }

      &.quiet {
        padding: unset;
        border: unset;
      }
    }
  }

  .form-tabs.isNew {
    width: 97%;
    gap: 0rem;
    display: flex;
    color: var(--spectrum-global-color-gray-600);
    justify-content: center;
    border-radius: 8px;
    border: 1px solid var(--spectrum-global-color-gray-300);
    animation: barberpole 1s linear infinite;
    background: repeating-linear-gradient(
      -45deg,
      var(--spectrum-global-color-gray-75),
      var(--spectrum-global-color-gray-75) 16px,
      var(--spectrum-global-color-gray-200) 16px,
      var(--spectrum-global-color-gray-200) 32px
    );
    overflow: hidden;

    & > .tab {
      height: 1rem;
      font-size: 11px;
      flex: auto;
      display: flex;
      justify-content: center;
      align-items: center;
      border: none;
      position: relative;
      border-top-right-radius: 8px;
      border-bottom-right-radius: 8px;

      &:hover {
        color: var(--spectrum-global-color-blue-400);
        background-color: transparent;
      }

      &.active {
        background-color: var(--spectrum-global-color-gray-300);
        color: var(--spectrum-global-color-gray-900);
        cursor: default;
        font-weight: 800;
      }

      &.complete {
        color: var(--spectrum-global-color-gray-700);
        background-color: var(--spectrum-global-color-gray-300);
        border-radius: unset;
      }
    }
  }

  @keyframes barberpole {
    from {
      background-position: 0 0;
    }
    to {
      background-position: 48px 0px;
    }
  }

  .form-tabs {
    display: flex;
    gap: 0.25rem;
    color: var(--spectrum-global-color-gray-600);
    margin-bottom: -1px;
    z-index: 1;

    & > .tab {
      width: 8rem;
      height: 1.85rem;
      display: flex;
      justify-content: center;
      align-items: center;
      border: 1px solid var(--spectrum-global-color-gray-300);
      border-top-left-radius: 4px;
      border-top-right-radius: 4px;
      border-bottom: 1px transparent;
      cursor: pointer;

      &:hover {
        background-color: var(--spectrum-global-color-gray-100);
      }

      &.active {
        background-color: var(--spectrum-global-color-gray-100);
        border-color: var(--spectrum-global-color-gray-300);
        color: var(--spectrum-global-color-gray-700);
        cursor: default;
        font-weight: 800;
        border-bottom: 1px solid var(--spectrum-global-color-gray-100);
      }
    }
  }
</style>
