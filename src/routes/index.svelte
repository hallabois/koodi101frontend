<script lang="ts">
    import { onMount } from "svelte";
    let dataPromise;
    onMount(async () => {
        fetchAccelData();
    } );
    async function fetchAccelData() {
        dataPromise = await fetch("http://95.216.143.30:9000/api/acceleration");
        console.log(dataPromise);
        return await dataPromise.json();
    }
</script>

<h1>Welcome to SvelteKit</h1>
<p>Visit <a href="https://kit.svelte.dev">kit.svelte.dev</a> to read the documentation</p>

{#await fetchAccelData()}
    <p>Ladataan dataa...</p>
{:then data} 
    {JSON.stringify(data)}
{/await}