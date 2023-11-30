<script lang="ts">
  import { onMount, afterUpdate } from 'svelte';

  export let input: string = "";
  export let isPremium: boolean = true;
  export let isVip: boolean = true

  let textElement: HTMLElement;
  let wrapperElement: HTMLElement;

  let fontSize = 64;
  const horizontalPadding = 20 * 2;

  onMount(() => {
    adjustFontSize();
  });

  afterUpdate(() => {
    adjustFontSize();
  });

  function adjustFontSize() {
    let parentWidth = wrapperElement.clientWidth - horizontalPadding;

    fontSize = 64;
    textElement.style.fontSize = `${fontSize}px`;
    while (textElement.scrollWidth > parentWidth && fontSize > 16) {
      fontSize--;
      textElement.style.fontSize = `${fontSize}px`;
    }
  }

</script>

<input type="text" bind:value={input} placeholder="Seed" maxlength="45" />
<div class="wrapper" bind:this={wrapperElement} style="max-width: 600px;">
  <div class="content">
    <slot></slot>
  </div>
  <div class="text-wrapper" style="background-color: {isPremium ? '#dffe00' : '#000'}; color: {isPremium ? '#000' : '#fff'}">
    <p bind:this={textElement} style="font-size: {fontSize}px">{input ? input + ".mode" : "" + ".mode"}</p>
  </div>
</div>

<style>
  
    div {
      background-color: #000;
      color: #fff;
      text-align: center;
      font-weight: 800;
      line-height: 100%;
    }

    .wrapper {
      border-radius: 20px;
      overflow: hidden;
    }

    .content {
      height: 600px;
      width: 600px;
    }

    .text-wrapper {
      min-height: 120px;
      display: flex; 
      justify-content: flex-end; 
      align-items: center; 
      padding: 0 20px; 
      max-width: 600px;
      white-space: nowrap;
      text-transform: lowercase;
      line-height: 100%;
    }
  
</style>