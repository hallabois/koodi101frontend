<script lang="ts">
    import { onMount } from "svelte";
    import Line from "svelte-chartjs/src/Line.svelte";

    const backend_url = "https://koodi101backend.herokuapp.com";

    let use_scuffed_smoothing = false;
    let smoothness = 2;
    let rangemin;
    let rangemax;
    let range_start;
    let range_end;
    function averageDataPoints(o1, o2) {
        return {
            ...o1,
            x: (o1.x + o2.x)/2,
            y: (o1.y + o2.y)/2,
            z: (o1.z + o2.z)/2
        }
    }
    function smoothData(dat: Array<any>, iter: number) {
        let data = dat;
        for(let i = 0; i < iter;i++) {
            data = data.map( (o, i)=>data[i+1] ? averageDataPoints(o, data[i+1]) : o );
            data = data.map( (o, i)=>data[i-1] ? averageDataPoints(o, data[i-1]) : o );
        }
        return data;
    }
    function smoothData2(ogdat: Array<any>, dat: Array<any>, iter: number) {
        let data = dat;
        for(let i = 0; i < iter;i++) {
            data = data.map( (o, i)=>ogdat[(i*iter)+1] ? averageDataPoints(o, ogdat[(i*iter)+1]) : o );
            data = data.map( (o, i)=>ogdat[(i*iter)-1] ? averageDataPoints(o, ogdat[(i*iter)-1]) : o );
        }
        return data;
    }
    function getRangeMinMax(dat: Array<any>) {
        let mapped = dat.map(x=>new Date(x.createdAt).getTime());
        let min = Math.min(...mapped);
        let max = Math.max(...mapped);
        return [min, max];
    }
    $: if(dataJSON && smoothness) {
        let ogData = dataJSON.results;
        [rangemin, rangemax] = getRangeMinMax(ogData);
        range_start = range_start || rangemin;
        range_end = range_end || rangemax;
        let filteredData = ogData.filter( (o)=>new Date(o.createdAt).getTime()>=range_start && new Date(o.createdAt).getTime()<=range_end );
        if(smoothness != 0){ // Modulo might not like 0
            if(use_scuffed_smoothing){
                filteredData = smoothData2(filteredData, filteredData.filter( (_o, i)=>i%smoothness==0 ), smoothness);
            }
            else {
                filteredData = smoothData(filteredData, smoothness).filter( (_o, i)=>i%smoothness==0 );
            }
        }
        let labels = filteredData.map( o=>o.id );
        let datax = filteredData.map( o=>o.x );
        let datay = filteredData.map( o=>o.y );
        let dataz = filteredData.map( o=>o.z );
        let datasets = [
            {
                label: "x",
                borderColor: "red",
                fill: false,
                data: datax
            },
            {
                label: "y",
                borderColor: "green",
                fill: false,
                data: datay
            },
            {
                label: "z",
                borderColor: "blue",
                fill: false,
                data: dataz
            },
        ];
        dataline = {
            labels: labels,
            datasets: datasets
        };
    }
    async function fetchAccelData() {
        dataPromise = await fetch(`${backend_url}/api/acceleration`);
        console.log(dataPromise);
        dataJSON = await dataPromise.json();
        return dataJSON;
    }
    let dataJSON;
    let dataline;
    let dataPromise;
    onMount(async () => {
        fetchAccelData();
    } );
</script>
<main>
    <div class="mainbox">
    <h1 class="otsikko">Koodi-101 - Kurssiprojekti</h1>
    <p class="teksti"> Elias, Eero, Joonas, Vinski</p>
    <p class="teksti">Visit <a href="https://kit.svelte.dev">kit.svelte.dev</a> to read the documentation</p>
    <p class="teksti">Käyttää kans <a href="https://www.chartjs.org/docs/latest/" target="_blank">charts.js</a>:sää</p>

    {#await fetchAccelData()}
        <p>Ladataan dataa...</p>
    {:then data} 
        {#if dataline}

            <p class="teksti">Original data was from {new Date(rangemin).toLocaleString()} to {new Date(rangemax).toLocaleString()}, 
                mapped to <b>{new Date(range_start).toLocaleString()} – {new Date(range_end).toLocaleString()}</b></p>
        <div class="graph">
            <div>
                <Line data={dataline} options={{
                    animation: false,
                    plugins: {
                        title: {
                            display: true,
                            text: 'Kiihtyvyys'
                        }
                    }
                }} />
            </div>
            <label for="smoothness">Käyrän tarkkuus</label>
            <input id="smoothness" bind:value={smoothness} type="range" min=1 max=100 />
            <input id="smoothness" bind:value={smoothness} type="number" min=1 max=100 />|
            <!-- <label for="smoothingalgo">Käytä rikkinäistä juttua</label> -->
            <!-- <input id="smoothness" bind:checked={use_scuffed_smoothing} type="checkbox" />| -->
            <label for="rangestart">Aikavälin alku</label>
            <input id="smoothness" bind:value={range_start} type="range" min={rangemin} max={Math.min(rangemax, range_end)} />|
            <label for="rangestart">Aikavälin loppu</label>
            <input id="smoothness" bind:value={range_end} type="range" min={Math.max(rangemin, range_start)} max={rangemax} />|
        </div>
        {#if data.results}
        <details>
            <summary>Raw data</summary>
            {#each data.results as result}
                {JSON.stringify(result)}
            {/each}
        </details>
        {/if}
    {/if}
{/await}
</main>

<style>
    main,:global(body,main) {
        background-color: #333;
    }
    .graph {
        max-width: 66%;
        margin: 0 auto;
    }

    .otsikko {
        font-size: 3.5em;
        text-align: center;
    }

    .teksti {
        text-align: center;
    }

    .mainbox {
        background-color: #1e2f3e;
        margin: 3em;
        padding: 0.8em;
        border-radius: 20px;
    }

    * {
        color: white ;
        font-family: 'Segoe UI', Tahoma, -apple-system, Roboto, Oxygen, Ubuntu, Cantarell, 'Open Sans', 'Helvetica Neue', sans-serif, Verdana, sans-serif;
    }
</style>