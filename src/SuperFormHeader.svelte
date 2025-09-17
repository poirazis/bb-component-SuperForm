<script>
  import { SuperButton } from "@poirazis/supercomponents-shared";
  import { getContext } from "svelte";

  // Props
  export let title;
  export let titleIcon;
  export let subtitle;
  export let formButtons = [];
  export let hideDefaultActions = false;
  export let disabled = false;
  export let quiet = false;
  export let multiStep = false;
  export let actionType = "create";
  export let multiRow = false;
  export let actionsMode;
  export let canDelete = false;
  export let canEdit = false;
  export let viewToEdit = false;
  export let isDirty = false;
  export let hasDataSource = false;
  export let richSteps = [];
  export let activeStep = 0;
  export let isParent = false;
  export let buttonPosition = "top";

  // Context
  const allContext = getContext("context");
  const { enrichButtonActions } = getContext("sdk");

  // Events
  import { createEventDispatcher } from "svelte";
  const dispatch = createEventDispatcher();

  // Computed
  $: size = quiet ? "S" : "M";
  $: buttons = generateButtons($$props);

  // Methods
  const handleButtonClick = (button) => (event) => {
    if (button.text === "Save" && isParent) {
      onSave();
    } else if (button.onClick) {
      const enrichedAction = enrichButtonActions(button.onClick, $allContext);
      if (enrichedAction) enrichedAction();
    }
  };

  const generateButtons = () => {
    const buttons = [];

    // Custom form buttons
    formButtons.forEach((button) => {
      buttons.push({
        ...button,
        disabled,
        onClick: handleButtonClick(button),
      });
    });

    // Default actions when hasDataSource and not hidden
    if (hasDataSource && !hideDefaultActions) {
      if (
        multiStep &&
        actionType === "create" &&
        richSteps[activeStep]?.buttons
      ) {
        richSteps[activeStep].buttons.forEach((button) => {
          buttons.push({
            ...button,
            onClick: handleButtonClick(button),
          });
        });
      } else if (!multiRow && actionType === "update" && canDelete) {
        buttons.push({
          icon: "ri-delete-bin-line",
          type: "warning",
          size,
          quiet: true,
          text: "Delete",
          onClick: () => dispatch("delete"),
          disabled,
        });
      }

      if (multiRow && actionsMode === "batch") {
        buttons.push({
          icon: "ri-save-line",
          text: "Save All",
          type: "cta",
          quiet: true,
          size,
          disabled,
          onClick: () => dispatch("saveMany"),
        });
      } else if (!multiRow && !multiStep) {
        if (actionType === "create") {
          buttons.push({
            text: "Reset",
            type: "secondary",
            size,
            quiet: true,
            disabled,
            onClick: () => dispatch("reset"),
          });
        }
        if (canEdit && actionType == "view") {
          buttons.push({
            icon: "ri-edit-line",
            type: "secondary",
            size,
            quiet: true,
            disabled,
            onClick: () => dispatch("toggleEdit"),
          });
        }

        if (viewToEdit) {
          buttons.push({
            text: "Cancel",
            type: "secondary",
            quiet: true,
            size,
            icon: actionType === "view" ? "ri-edit-line" : null,
            disabled,
            onClick: () => dispatch("toggleEdit"),
          });
        }
        if (actionType !== "view") {
          buttons.push({
            icon: "ri-save-line",
            text: "Save",
            disabled,
            size,
            type: isDirty ? "cta" : "primary",
            onClick: () => dispatch("save"),
          });
        }
      }
    }

    return buttons;
  };
</script>

<div class="form-header" class:bottom={buttonPosition == "bottom"}>
  <div class="form-title">
    <span class="title" class:small={subtitle || quiet}>
      {#if titleIcon}
        <i class={titleIcon} />
      {/if}
      {title}
    </span>

    {#if subtitle}
      <span class="sub-title">{subtitle}</span>
    {/if}
  </div>
  <div class="form-buttons">
    {#each buttons as button}
      <SuperButton {...button} />
    {/each}
  </div>
</div>

<style>
  .form-header {
    display: flex;
    flex-direction: row;
    justify-content: space-between;
    align-items: center;
    min-height: 2.4rem;
    padding-bottom: 0.75rem;
    border-bottom: 1px solid var(--spectrum-global-color-gray-300);

    &.bottom {
      border-top: 1px solid var(--spectrum-global-color-gray-300);
      border-bottom: unset;
      padding-top: 0.75rem;
      padding-bottom: unset;
      order: 1 !important;

      & > .form-title {
        visibility: hidden;
      }
    }
  }

  .form-title {
    display: flex;
    flex-direction: column;
  }

  .title {
    font-size: 17px;
    font-weight: 600;
    display: flex;
    align-items: center;
    gap: 0.5rem;
    opacity: 0.8;
  }

  .title.small {
    font-size: 14px;
  }

  .sub-title {
    line-height: 1rem;
    font-size: 12px;
    font-weight: 400;
    color: var(--spectrum-global-color-gray-600);
  }

  .form-buttons {
    display: flex;
    gap: 0.5rem;
    align-items: center;
  }
</style>
