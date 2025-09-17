<script>
  import { getContext, createEventDispatcher } from "svelte";
  import { SuperButton } from "@poirazis/supercomponents-shared";

  const { enrichButtonActions, builderStore } = getContext("sdk");
  const allContext = getContext("context");

  const form = getContext("form");

  const dispatch = createEventDispatcher();
  export let disabled;
  export let actionsMode;
  export let inView;
  export let compact;
  export let quiet;
  export let isNew;
  export let richSteps;
  export let canDelete;
  export let canAdd;
  export let multiRow;
  export let rowActionButtons;
  export let rowActions;
  export let comp_id;
  export let row;
  export let formObject;
  export let isDeleted;

  let activeStep;

  $: inBuilder = $builderStore.inBuilder;
  $: multiStep = richSteps?.length > 1;
  $: buttons = generateButtons(rowActions, rowActionButtons, dirty, activeStep);
  $: formState = form?.formState;
  $: dirty = $formState?.dirty;
  $: unsub = form?.formState?.subscribe(
    (v) => (activeStep = Math.max(v.currentStep - 1, 0))
  );
  $: onStepChange(activeStep);

  const generateButtons = () => {
    const buttons = [];

    // Add row action buttons
    if (rowActionButtons?.length) {
      buttons.push(
        ...rowActionButtons.map(({ onClick, ...rest }) => ({
          ...rest,
          size: "S",
          onClick: () => handleRowAction(onClick),
        }))
      );
    }

    // Add row actions
    if (rowActions) {
      buttons.push(
        ...Object.entries(rowActions).map(([name, action]) => ({
          icon: "ri-flashlight-line",
          type: "primary",
          quiet: true,
          size: "S",
          text: action.name,
        }))
      );
    }

    // Add action buttons when not in view
    if (!inView) {
      if (isDeleted) {
        buttons.push({
          icon: "ri-arrow-go-back-fill",
          size: "S",
          quiet: true,
          onClick: onDelete,
        });
      } else if (canDelete) {
        buttons.push({
          icon: "ri-delete-bin-2-line",
          type: "warning",
          size: "S",
          disabled,
          fillOnHover: true,
          quiet: true,
          onClick: onDelete,
        });
      }

      if (dirty) {
        buttons.push({
          icon: "ri-refresh-line",
          size: "S",
          type: "secondary",
          quiet: true,
          disabled,
          fillOnHover: true,
          onClick: onReset,
        });
      }

      if (actionsMode === "individual") {
        buttons.push({
          icon: "ri-save-line",
          disabled,
          size: "S",
          type: "primary",
          quiet: true,
          onClick: onSave,
        });
      }

      if (canAdd) {
        buttons.push({
          icon: "ri-add-line",
          size: "S",
          type: "primary",
          disabled: !canAdd || disabled,
          fillOnHover: true,
          quiet: true,
          onClick: () => dispatch("add-row", {}),
        });
      }
    }

    return buttons;
  };

  const onDelete = () => {
    dispatch("delete", {});
  };

  const onSave = () => {
    dispatch("save", {});
  };

  const onReset = () => {
    dispatch("reset", {});
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

  const onStepChange = (step) => {
    dispatch("changeStep", step);
  };
</script>

<!-- svelte-ignore a11y-no-noninteractive-tabindex -->
<!-- svelte-ignore a11y-no-static-element-interactions -->
<!-- svelte-ignore a11y-mouse-events-have-key-events -->
<!-- svelte-ignore a11y-click-events-have-key-events -->
<div class="row-wrapper" class:multiRow class:dirty class:compact>
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
              class:complete={activeStep > idx}
              class:active={idx == activeStep}
              on:click={() => {
                if (inBuilder) return;
                if (form.formApi.validate()) {
                  form.formApi.setStep(idx + 1);
                }
              }}
            >
              {title}
            </div>
          {/each}
        </div>
      {/if}
      {#if multiRow && buttons.length}
        <span class="row-buttons">
          {#each buttons as button}
            <SuperButton {...button} on:click={button.onClick} />
          {/each}
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

      &.dirty {
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
        margin-bottom: 8px;
        border-bottom: 1px solid var(--spectrum-global-color-gray-300);

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
