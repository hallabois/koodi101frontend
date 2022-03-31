<script lang="ts">
    import { onMount } from "svelte";
    import Line from "svelte-chartjs/src/Line.svelte";

    const backend_url = "https://koodi101backend.herokuapp.com";

    let use_scuffed_smoothing = false;
    let smoothness = 1;
    let rangemin;
    let rangemax;
    let range_start;
    let range_start_offset = 5 * 60 * 1000;
    let range_end;
    $: smoothness = Math.floor((range_start_offset / 60 / 1000) /10);
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
    $: if( smoothness != null && range_start_offset != null && dataJSON ) {
        let ogData = dataJSON.results;
        [rangemin, rangemax] = getRangeMinMax(ogData);
        range_end = rangemax;
        range_start = range_end - range_start_offset || rangemin;
        let filteredData = ogData.filter( (o)=>new Date(o.createdAt).getTime()>=range_start && new Date(o.createdAt).getTime()<=range_end );
        if(smoothness != 0){ // Modulo might not like 0
            if(use_scuffed_smoothing){
                filteredData = smoothData2(filteredData, filteredData.filter( (_o, i)=>i%smoothness==0 ), smoothness);
            }
            else {
                filteredData = smoothData(filteredData, smoothness).filter( (_o, i)=>i%smoothness==0 );
            }
        }
        let labels = filteredData.map( o=>new Date(o.createdAt).toLocaleTimeString() );
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

    async function fetchTemps() {
        tempPromise = await fetch (`${backend_url}/api/temperature`);
        console.log(tempPromise);
        tempJSON = await tempPromise.json();
        return tempJSON;
    }
    let tempPromise;
    let tempJSON;

    onMount(async () => {
        fetchAccelData();
        fetchTemps();
        setInterval(()=>{
            console.log("refreshing...");
            fetchAccelData();
            fetchTemps();
        }, 10000 );
    } );

</script>
<main>
    <div class="mainbox">
    <h1 class="otsikko">Koodi-101 - Kurssiprojekti</h1>
    <p class="teksti"> Elias, Eero, Joonas, Vinski</p>

    {#if tempJSON}
        <p class="teksti spacialbs">Viimeisin lämpötila: {tempJSON.results.reduce((best,x)=>x.createdAt < best ? x : best).temp}℃</p>
    {/if}
    {#if dataJSON}
        {#if dataline}
            <p class="teksti spacialbs">Kiihtyvyys:</p>
            <p class="teksti">Original data was from {new Date(rangemin).toLocaleString()} to {new Date(rangemax).toLocaleString()}, 
                mapped to <b>{new Date(range_start).toLocaleString()} – {new Date(range_end).toLocaleString()}</b></p>
            <div class="graph">
                <div class="graphdata">
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
                <div class="control">
                    <label class="flex" for="smoothness">Käyrän tarkkuus</label>
                    <input class="flex" id="smoothness" bind:value={smoothness} type="range" min=1 max=100 />
                    <input class="flex" id="smoothness" bind:value={smoothness} type="number" min=1 max=100 />
                    <!-- <label for="smoothingalgo">Käytä rikkinäistä juttua</label> -->
                    <!-- <input id="smoothness" bind:checked={use_scuffed_smoothing} type="checkbox" />| -->
                    <label class="flex" for="rangestart">Aikaväli</label> {range_start_offset / 60 / 1000} min
                    <input class="flex" id="smoothness" bind:value={range_start_offset} type="range" min={0} max={rangemax-rangemin} />
                    <!-- <label class="flex" for="rangestart">Aikavälin loppu</label>
                    <input class="flex" id="smoothness" bind:value={range_end} type="range" min={Math.max(rangemin, range_start)} max={rangemax} /> -->
                </div>
            </div>
            {#if dataJSON.results && tempJSON.results}
                <details>
                    <summary>Raw data</summary>
                    {#each dataJSON.results as result}
                        {JSON.stringify(result)}
                        <br />
                    {/each}

                    {#each tempJSON.results as result}
                        {JSON.stringify(result)}
                        <br />
                    {/each}
                </details>
            {/if}
        {/if}
    {:else}
        <p>Ladataan dataa...</p>
    {/if}
</main>

<style>
    main,:global(body,main) {
        background-color: #333;
        margin:0;
    }
    .graph {
        color: black !important;
        max-width: 66%;
        margin: 0 auto;
        background: #ececec;
        padding: 1em 3em;
        border-radius: 0.3em;
    }

    .graphdata {
        padding: 1em;
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
        margin: 1em;
        padding: 0.8em;
        border-radius: 20px;
    }

    label {
        color: black;
        margin: 0em 1em 0;
    }

    .control {
        flex: auto;
        color: black;
    }

    input {
        color: black;
    }

    .flex {
        display: flex;
    }

    .spacialbs {
        font-size: 2.8em;
        font-family: serif;
    }


    * {
        color: white ;
        font-family: 'Segoe UI', Tahoma, -apple-system, Roboto, Oxygen, Ubuntu, Cantarell, 'Open Sans', 'Helvetica Neue', sans-serif, Verdana, sans-serif;
    }
</style>