<script context="module">
	export async function preload({ params }) {
     // the `slug` parameter is available because
     // this file is called [slug].svelte
     return { lineage: params.slug };
	}
</script>

<script>

  import Icon from 'svelte-awesome/components/Icon.svelte'
  import { warning } from 'svelte-awesome/icons';
  import { onMount } from "svelte";
	import * as R from "ramda";
  import config from "../../../config.json";

	export let lineage;
	let dagData = [];
	let taskData = [];
  let groupedTaskData = {'': {}};

  let genes = ["nsp3","nsp4","3C","nsp6","nsp7","nsp8","nsp9","nsp10","helicase","exonuclease","endornase","S","E","M","N","ORF3a","ORF6","ORF7a","ORF8","RdRp","methyltransferase"]
  let methods = ["meme_full","fade","fel","relax","bgm","busted","meme","prime","cfel","absrel","slac"]

  let methodDict = {
      "meme_full" : {
          "vizLink": "meme",
          "fullName": "MEME Full",
      },
      "fade" : {
          "vizLink": "fade",
          "fullName": "FADE",
      },
      "fel" : {
          "vizLink": "fel",
          "fullName": "FEL",
      },
      "relax" : {
          "vizLink": "relax",
          "fullName": "RELAX",
      },
      "bgm" : {
          "vizLink": "bgm",
          "fullName": "BGM",
      },
      "busted" : {
          "vizLink": "busted",
          "fullName": "BUSTED",
      },
      "meme" : {
          "vizLink": "meme",
          "fullName": "MEME",
      },
      "prime" : {
          "vizLink": "prime",
          "fullName": "PRIME",
      },
      "cfel" : {
          "vizLink": "contrast-fel",
          "fullName": "Contrast-FEL",
      },
      "absrel" : {
          "vizLink": "absrel",
          "fullName": "aBSREL",
      },
      "slac" : {
          "vizLink": "slac",
          "fullName": "SLAC",
      }
  };

  // selection_analyses_nsp2.meme_full
  let taskNames = genes.flatMap(d => methods.map(v => "selection_analyses_" + d + "." + v))

  function getStateBGColor(task) {
    if(task.state == "success") {
      return "bg-green-400"
    } else {
      return "bg-red-400"
    }
  }

  onMount(async () => {

    let url = "http://silverback.temple.edu/veg/airflow/api/v1/dags/rascl_" + lineage + "/dagRuns"
    let queryUrl = url + "?order_by=execution_date"
    let username = config.apiUserName;
    let password = config.apiPassword;
    let dagRunId = null;

    let ignoredStates = ['running', 'failed', 'queued', 'none']

    // Get dagRun id of latest completed run (state not running)

    let headers = new Headers();
    headers.set('Authorization', 'Basic ' + btoa(username + ":" + password));

    const res = await fetch(queryUrl, {
        method:'GET',
        headers: headers,
       });

    const data = await res.json();

    if (res.status === 200) {

      dagData = R.reject(d => R.includes(d.state, ignoredStates), data.dag_runs)

      if(R.length(dagData)) {
        dagRunId = R.last(dagData).dag_run_id;
      }

    } else {
      console.log(res.status, data.message);
      return;
    }

    if (!dagRunId) {
      return;
    }

    let dagRunUrl = url + "/" + dagRunId + "/taskInstances?limit=500";

    const taskRes = await fetch(dagRunUrl, {
        method:'GET',
        headers: headers,
       });

    const taskRawData = await taskRes.json();

    if (taskRes.status === 200) {

      taskData = R.filter(d=> R.includes(d.task_id, taskNames), taskRawData.task_instances)
      // Create results url
      // http://data.hyphy.org/web/covid-19/selection-analyses/rascl/AY.12/scheduled__2021-08-22T00%3A00%3A00%2B00%3A00/sequences.N.MEME.json
      let baseResultsUrl = "http://data.hyphy.org/web/covid-19/selection-analyses/rascl/" + lineage + "/" + dagRunId + "/";

      R.forEach(d => {
          let [gene, method] = d.task_id.split('_')[2].split('.');
          d.resultsUrl = baseResultsUrl + 'sequences.' + gene + '.' + R.toUpper(method) + '.json';
          d.hyphyVisionUrl = "http://vision.hyphy.org/" + methodDict[method].vizLink + "?resultsUrl=" + encodeURIComponent(d.resultsUrl);
          d.gene = gene;
          d.method = method;
      }, taskData);

      groupedTaskData = R.groupBy(d => d.gene, taskData);
      groupedTaskData = R.map(g => R.indexBy(d => d.method, g), groupedTaskData)
      console.log(groupedTaskData);

    } else {

      console.log(taskRes.status, taskRawData.message);
      return;

    }

  });

</script>

<style global lang="postcss">

	h1 {
		text-align: center;
		margin: 0;
  }

	h2 {
		text-align: center;
		margin: 0;
	}

	h1 {
		font-size: 2.8em;
		text-transform: uppercase;
		font-weight: 700;
		margin: 0 0 0.5em 0;
	}

  h2 {
		font-size: 1.4em;
		text-transform: uppercase;
		font-weight: 100;
		margin: 0 0 0.5em 0;
	}

	@media (min-width: 480px) {
		h1 {
			font-size: 4em;
		}
	}

</style>

<svelte:head>
	<title>Lineage</title>
</svelte:head>

<h2>{ lineage }</h2>

{#if !R.length(dagData) }
<div class="flex w-full h-full p-2 bg-yellow-400">
  <div class="text-center w-full"> <Icon data={warning} /> No Successful Analyses For This Lineage <Icon data={warning} /></div>
</div>
{/if}

<h1>Summary</h1>

<h1>Individual Results</h1>

<div class="content">
  {#each Object.keys(groupedTaskData).sort() as group}
    <h2>{ group }</h2>
    <div class="grid gap-4 grid-cols-3">
      {#each R.sortBy(R.prop('method'))(Object.values(groupedTaskData[group])) as task}
        <div class="grid grid-rows-2 grid-flow-col place-items-center { getStateBGColor(task) }">
          <div class="col-span-2 bg-white text-center w-full h-full p-2">{ methodDict[task.method].fullName }</div>
          {#if task.state == "success"}
            <div class="row-span-1 text-center w-full border-r-2 border-black"><a href={task.hyphyVisionUrl}>Visualization</a></div>
            <div class="row-span-1 text-center w-full"><a href={task.resultsUrl}>Export</a></div>
          {:else}
            <div class="col-span-2 row-span-1 text-center w-full">Not Available</div>
          {/if}
        </div>
      {/each}
    </div>
  {/each}

</div>

