<script>
  import { getContext } from "svelte";
  import { SuperForm } from "@poirazis/supercomponents-shared";

  export let text;

  const { styleable } = getContext("sdk");
  const component = getContext("component");
</script>

<div use:styleable={$component.styles}>
  <SuperForm {...$$props}>
    <slot />
  </SuperForm>
</div>
