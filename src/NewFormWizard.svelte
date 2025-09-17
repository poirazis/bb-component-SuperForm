<script>
  import { createEventDispatcher } from "svelte";

  export let nested = false;
  export let subformType = "link";
  export let outerFormJson = [];
  export let outerFormEmbed = [];
  export let dataSource = {};

  const dispatch = createEventDispatcher();

  // Dispatch events for field selections
  const handleFieldSelect = (field, type) => {
    dispatch("selectField", { field, subformType: type });
  };

  // Determine if the schema is invalid for link subform
  $: isInvalidSchema = dataSource.type !== "link";
</script>

<!-- svelte-ignore a11y-click-events-have-key-events -->
<!-- svelte-ignore a11y-no-static-element-interactions -->
<div class="empty" class:nested>
  {#if !nested}
    Super Form Pro
  {:else}
    <h3>
      Subform Type - {subformType === "json"
        ? "JSON Column"
        : subformType === "embed"
          ? "Embed Document"
          : "Relationship"}
    </h3>
    <div class="form-types">
      {#if subformType === "json"}
        <div class="form-type">
          JSON Columns
          {#if outerFormJson?.length}
            {#each outerFormJson as field}
              <div
                class="field"
                on:click={() => handleFieldSelect(field, "json")}
              >
                <i class="ri-braces-line" />
                {field}
              </div>
            {/each}
          {:else}
            <div class="field">No JSON fields available</div>
          {/if}
        </div>
      {:else if subformType === "embed"}
        <div class="form-type">
          Embed Document In
          {#if outerFormEmbed?.length}
            {#each outerFormEmbed as field}
              <!-- svelte-ignore a11y-click-events-have-key-events -->
              <div
                class="field"
                on:click={() => handleFieldSelect(field, "embed")}
              >
                <i class="ri-braces-line" />
                {field}
              </div>
            {/each}
          {:else}
            <div class="field">No embed fields available</div>
          {/if}
        </div>
      {:else}
        <div class="form-type" class:invalid={isInvalidSchema}>
          {#if isInvalidSchema}
            Invalid Schema Selected
          {:else}
            Pick From Schema
          {/if}
        </div>
      {/if}
    </div>
  {/if}
</div>

<style>
  .empty {
    flex: auto;
    display: flex;
    flex-direction: column;
    align-items: stretch;
    gap: 0.25rem;
    padding: 1rem;
    border: 2px dashed var(--spectrum-global-color-gray-400);
  }

  .empty.nested {
    border: 1px dashed var(--spectrum-global-color-blue-400);
  }

  .form-types {
    padding: 0.5rem;
    display: grid;
    grid-template-columns: repeat(1, 1fr);
    align-items: stretch;
    gap: 1rem;
  }

  .form-type {
    flex: auto;
    display: flex;
    flex-direction: column;
    border: 1px solid var(--spectrum-global-color-gray-300);
    background-color: var(--spectrum-global-color-gray-100);
    border-radius: 4px;
    gap: 0.5rem;
    padding: 0.5rem;
  }

  .form-type.invalid {
    color: var(--spectrum-global-color-red-400);
    border-color: var(--spectrum-global-color-red-500);
  }

  .form-type > .field {
    display: flex;
    align-items: center;
    gap: 0.5rem;
    padding-left: 0.5rem;
    line-height: 2rem;
    border: 1px solid var(--spectrum-global-color-gray-300);
    background-color: var(--spectrum-global-color-gray-50);
  }

  .form-type > .field:hover {
    background-color: var(--spectrum-global-color-blue-100);
    border: 1px solid var(--spectrum-global-color-blue-500);
    cursor: pointer;
  }
</style>
