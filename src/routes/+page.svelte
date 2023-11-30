<script lang="ts">
    import Wrapper from "$lib/components/wrapper.svelte"
    import Scene from  "$lib/components/scene.svelte"

    let input: string = ""
    let isPremium: boolean = false
    let isVip: boolean = true

</script>


<Wrapper bind:input={input} bind:isPremium={isPremium} bind:isVip={isVip}>
    <Scene bind:input={input} bind:isPremium={isPremium}  bind:isVip={isVip}/>
</Wrapper>
