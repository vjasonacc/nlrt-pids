<script lang="ts">
  import { onMount } from 'svelte';
  import { getApiURL, ArrivalEntry } from '../../data';

  export let stnId : string;
  export let targetPlatform : number;
  let etaData: ArrivalEntry[] = [];
  const DisplayedArrivalRow = 10;

  async function fetchData(stnId: string, targetPlatform: number, fallback: boolean = false) {
    if(stnId == null || targetPlatform == null) return;
    let resp;
    try {
      resp = await fetch(getApiURL(stnId, fallback));
    } catch {
      if(!fallback) {
        console.warn("Cannot fetch data from proxy server, falling back to MTR")
        return fetchData(stnId, targetPlatform, true);
      }
    }

    let data = await resp.json();

    /* Only status 1 should be valid */
    if(data.status != 1) return;

    let entries: ArrivalEntry[] = [];
    for(let platform of data.platform_list) {
      if(platform.platform_id !== targetPlatform) continue;
      
      for(let rawEntry of platform.route_list) {
          /* Line stopped */
          if(rawEntry.stop === 1) continue;
          let timetext: string;
          let timenum: string;

          if(rawEntry.time_en.includes("min")) {
            timetext = "分鐘|min";
            timenum = rawEntry.time_en.split(" min")[0];
          } else if (rawEntry.time_en === "-") {
            timetext = "\xa0|\xa0";
            timenum = "-";
          } else {
            timenum = "";
            timetext = `${rawEntry.time_ch}|${rawEntry.time_en}`;
          }

          let entry = new ArrivalEntry(rawEntry.route_no, rawEntry.train_length, rawEntry.dest_ch, rawEntry.dest_en, timenum, timetext);
          entries.push(entry);
      }
    }
    etaData = entries;
  }

  $: fetchData(stnId, targetPlatform);

  onMount(async () => {
    const fetchInterval = setInterval(async() => await fetchData(stnId, targetPlatform), 10000);

    return () => clearInterval(fetchInterval);
  })
</script>

<main>
  <table id="eta">
      <tr class="header">
          <td><span class="sizeZH">路線</span><br>Route</td>
          <td><span class="sizeZH">目的地</span><br>Destination</td>
          <td></td>
          <td><span class="sizeZH">下班車</span><br>Next Train</td>
      </tr>

      {#each {length: DisplayedArrivalRow} as _, i}
        {#if i < etaData.length}
          <tr class="arrRow">
            <td class="lineHeightDouble sizeDouble">{etaData[i].route_no}</td>
            <td><span class="sizeZH">{etaData[i].dest_zh}</span><br>{etaData[i].dest_en}</td>
            <td>
              <div class="cars">
                {#each {length: etaData[i].train_length} as _, i}
                <img class="car" src="img/lrv.png" alt="LRV Car {i}">
                {/each}
              </div>
            </td>
            <td><span class="sizeDouble multiLine">{etaData[i].time_num}</span><span class="eta">{etaData[i].time_text.split("|")[0]}<br>{etaData[i].time_text.split("|")[1]}</span></td>
          </tr>
        {:else}
          <tr class="arrRow">
            <td class="lineHeightDouble sizeDouble">&nbsp;</td>
            <td></td>
            <td></td>
            <td></td>
          </tr>
        {/if}
      {/each}
  </table>
</main>

<style>
  @import url('https://fonts.googleapis.com/css2?family=Noto+Sans:wght@600&family=Noto+Serif+HK:wght@600&display=swap');
  :root {
      --font-family: "Myriad Pro", "Noto Sans", "Noto Serif HK";
      --aqua: #C7EBFB;
      --title-color: #153156;
  }

  main {
      background-color: #FFF;
      overflow-y: hidden;
  }

  table {
      border-spacing: 15px;
      width: 100%;
  }

  tr td {
      vertical-align: middle;
      padding: 5px;
  }

  #eta {
      font-size: 2.2vh;
      font-family: var(--font-family);
      font-weight: 600;
  }

  #eta > tr:nth-child(2n+1) {
      background-color: #FFF8F2;
      border: 1px solid lightgray;
  }

  #eta > tr > td {
      padding: 0 10px;
  }

  .header {
      font-family: var(--font-family);
      font-size: 1.5vh;
      padding: 5px;
  }

  .header td {
      background-color: lightgray;
  }

  .arrRow {
      vertical-align: middle;
      padding: 10px;
  }

  #eta {
      table-layout: fixed;
  }

  #eta > tr > td:nth-child(1) {
      width: 16%;
  }

  #eta > tr > td:nth-child(2) {
      max-width: 30%;
  }

  #eta > tr > td:nth-child(3) {
      text-align: right;
      white-space: nowrap;
      width: 11vh;
      position: relative;
  }

  .cars {
    position: absolute;
    right: 0;
    top: 50%;
    transform: translateY(-50%);
  }

  .car {
    width: 6vh;
  }

  #eta > tr > td:last-child {
      width: 11vh;
      text-align: right;
  }

  .multiLine {
    margin: 5px;
    line-height: 0em;
    vertical-align: bottom;
    width: 6vh;
  }

  .sizeZH {
      font-size: 1.45em;
  }

  .sizeDouble {
      font-size: 2em;
  }

  .lineHeightDouble {
      line-height: 2em;
  }
</style>