<script>
  import { getContext } from "svelte";
  import { SuperForm } from "@poirazis/supercomponents-shared";

  export let flex;
  export let onChange;
  export let onReset;
  export let actionType = "Create";
  export let dataSource = { name: "Custom", label: "Custom" };
  export let disabled = false;
  export let readonly = false;
  export let initialFormStep = 1;

  const { styleable, memo } = getContext("sdk");
  const component = getContext("component");

  $: $component.styles = {
    ...$component.styles,
    normal: {
      ...$component.styles.normal,
      flex,
    },
  };
</script>

<div use:styleable={$component.styles}>
  <SuperForm
    {actionType}
    {dataSource}
    {disabled}
    {readonly}
    {initialFormStep}
    on:change={onChange}
    on:reset={onReset}
  >
    <slot />
  </SuperForm>
</div>
