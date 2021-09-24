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
  import { onMount, beforeUpdate } from "svelte";
	import * as R from 'ramda';
	import * as d3 from 'd3';
  import * as Plot from '@observablehq/plot';
  import config from '../../../config.json';
  import Select from 'svelte-select';

	export let lineage;
  export let report;

	let dagData = [];
	let taskData = [];
  let groupedTaskData = {'': {}};
  let summaryReport = {};
  let summaryPlotItems = ['num_seqs', 'num_uniq_haps', 'tree_len_internal', 'tree_len_terminal', 'mean_dnds_leaf', 'mean_dnds_internal', 'num_sites', 'codon_num_var_sites', 'codon_num_var_sites_minor', 'prot_num_var_sites', 'prot_num_var_sites_minor', 'num_sites_pos_fel', 'num_sites_neg_fel', 'num_sites_pos_cfel', 'num_sites_neg_cfel', 'num_sites_directional', 'num_sites_coevolving', 'num_sites_meme', 'median_branches_meme'];
  let summaryPlotContainer;
  let plotSummarySelect = summaryPlotItems[0];

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

  function handleSummarySelect(item) {
    plotSummarySelect = item.detail.value;
    plotSummary(report, summaryPlotContainer)
  }

  async function getSummaryResults(reportUrl) {

    const report = await d3.csv(reportUrl);
    return report;

  }

  let taskNames = genes.flatMap(d => methods.map(v => "selection_analyses_" + d + "." + v))



  function getStateBGColor(task) {

    if(task.state == "success") {
      return "bg-green-400"
    } else {
      return "bg-red-400"
    }

  }

  function plotSummary(report, summaryPlotContainer) {

    d3.select(".plot-container").select('svg').remove();

    let width = 640;
    R.forEach( d => d[plotSummarySelect] = parseFloat( d[plotSummarySelect] ) || 0, report);

    let options = {
      marginTop: 50,
      marginLeft: 200,
      marginBottom: 50,
      width: width,
      y : {
        grid: true,
        domain: R.map(d => d.gene, R.sortBy(R.prop(plotSummarySelect))(report))
      },
      marks: [
        Plot.ruleX([0]),
        Plot.barX(report, {y: "gene", x: plotSummarySelect })
      ]
    };

    summaryPlotContainer.appendChild(Plot.plot(options));

  }

  function getSummaryInfo(report) {

    let medianSeqs = R.pipe(
                        R.map(d => parseInt(d.num_seqs)),
                        R.filter(d => d),
                        R.median
                       )(report);

    // TODO
    // let medianReferenceSeqs = R.median(R.map(d => parseInt(d.num_seqs), report));

    // Diversifying positive selection
    let diversifyingSites = R.pipe(
        R.map(d => parseInt(d.num_sites_pos_fel)),
        R.filter(d => d),
        R.sum
      )(report); 

    // TODO
    let diversifyingSitesAllBranches = R.pipe(
        R.map(d => parseInt(d.num_sites_pos_fel)),
        R.filter(d => d),
        R.sum
      )(report); 

    // Significant differences
    let sigDiff = R.pipe(
      R.map(d => parseInt(d.num_sites_pos_cfel)),
      R.filter(d => d),
      R.sum
    )(report);

    sigDiff += R.pipe(
      R.map(d => parseInt(d.num_sites_neg_cfel)),
      R.filter(d => d),
      R.sum
    )(report);

    // TODO: Out of X successful runs
    let directionalSelection = R.pipe(
      R.map(d => parseInt(d.num_sites_directional)),
      R.filter(d => d),
      R.sum
    )(report);

    let coevolvingPairs = R.pipe(
      R.map(d => parseInt(d.num_sites_coevolving)),
      R.filter(d => d),
      R.sum
    )(report);

    let purifyingSites = R.pipe(
      R.map(d => parseInt(d.num_sites_neg_fel)),
      R.filter(d => d),
      R.sum
    )(report);

    // TODO Episodic Branch Selection
    let episodicBranchSelection = null;

    return {
      medianSeqs: medianSeqs,
      diversifyingSites: diversifyingSites,
      diversifyingSitesAllBranches: diversifyingSitesAllBranches,
      sigDiff: sigDiff,
      directionalSelection: directionalSelection,
      coevolvingPairs: coevolvingPairs,
      purifyingSites: purifyingSites,
      episodicBranchSelection: episodicBranchSelection,
    }

  }

  onMount(async () => {

    let url = "http://silverback.temple.edu/veg/airflow/api/v1/dags/rascl_" + lineage + "/dagRuns";
    let queryUrl = url + "?order_by=execution_date";
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
        headers: headers
       });

    const taskRawData = await taskRes.json();

    if (taskRes.status === 200) {

      taskData = R.filter(d=> R.includes(d.task_id, taskNames), taskRawData.task_instances)

      // Create results url
      let baseResultsUrl = "http://data.hyphy.org/web/covid-19/selection-analyses/rascl/" + lineage + "/" + dagRunId + "/";

      // Reports
      let reportUrl = baseResultsUrl + "report.csv";
      report = await getSummaryResults(reportUrl);
      summaryReport = getSummaryInfo(report);

    // Report Plots
    plotSummary(report, summaryPlotContainer);


      R.forEach(d => {
          let [gene, method] = d.task_id.split('_')[2].split('.');

          // if BGM, the filename is different
          if(R.toUpper(method) == 'BGM') {
            d.resultsUrl = baseResultsUrl + 'sequences.' + gene + '.combined.fas.' + R.toUpper(method) + '.json';
          } else {
            d.resultsUrl = baseResultsUrl + 'sequences.' + gene + '.' + R.toUpper(method) + '.json';
          }
          d.hyphyVisionUrl = "http://vision.hyphy.org/" + methodDict[method].vizLink + "?resultsUrl=" + encodeURIComponent(d.resultsUrl);
          d.gene = gene;
          d.method = method;
      }, taskData);

      groupedTaskData = R.groupBy(d => d.gene, taskData);
      groupedTaskData = R.map(g => R.indexBy(d => d.method, g), groupedTaskData)

    } else {

      console.log(taskRes.status, taskRawData.message);
      return;

    }

  });

</script>

<style global lang="postcss">

  .plot-container {
    overflow-x: scroll; 
  }

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


<div class="grid gap-4 grid-cols-3">

  <div class="row-span-1 text-center w-full bg-green-400">
    <h3>Diversifying Positive Selection Along Internal Branches</h3>
    <div>{ summaryReport.diversifyingSites }</div>
  </div>

  <div class="row-span-1 text-center w-full bg-green-400">
    <h3>Diversifying Positive Selection Across All Branches</h3>
    <div>{ summaryReport.diversifyingSitesAllBranches }</div>
  </div>

  <div class="row-span-1 text-center w-full bg-green-400">
    <h3>Significant differences from reference sequences</h3>
    <div>{ summaryReport.sigDiff }</div>
  </div>

  <div class="row-span-1 text-center w-full bg-green-400">
    <h3>Directional Selection</h3>
    <div>{ summaryReport.directionalSelection }</div>
  </div>

  <div class="row-span-1 text-center w-full bg-green-400">
    <h3>Coevolution Pairs</h3>
    <div>{ summaryReport.coevolvingPairs }</div>
  </div>

  <div class="row-span-1 text-center w-full bg-green-400">
    <h3>Negative Selection</h3>
    <div>{ summaryReport.purifyingSites }</div>
  </div>

  <div class="row-span-1 text-center w-full bg-green-400">
    <h3>Episodic Branch Selection</h3>
    <div>{ summaryReport.episodicBranchSelection }</div>
  </div>

</div>

<div class="grid gap-4 grid-cols-2 items-center">
  <div bind:this={ summaryPlotContainer } class="plot-container"></div>
  <Select class="test" items={ summaryPlotItems } on:select={ handleSummarySelect } ></Select>
</div>

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

