<script>
  import { getContext } from "svelte";
  import { SuperForm } from "@poirazis/supercomponents-shared";

  export let flex;
  export let onChange;
  export let onReset;

  const { styleable, memo } = getContext("sdk");
  const component = getContext("component");

  let props = memo($$props);
  $: props.set($$props);

  $: $component.styles = {
    ...$component.styles,
    normal: {
      ...$component.styles.normal,
      flex,
    },
  };
</script>

<div use:styleable={$component.styles} class="bb-component-SuperFormSimple">
  <SuperForm {...$props} on:change={onChange} on:reset={onReset}>
    <slot />
  </SuperForm>
</div>
