<script context="module">
	export async function preload({ params }) {
     return { lineage: params.slug };
	}
</script>

<script>
  import { onMount } from "svelte";
	import * as R from "ramda";

	export let lineage;
	let dagData = [];
	let taskData = [];

  // selection_analyses_nsp2.meme_full
  let genes = ["nsp3","nsp4","3C","nsp6","nsp7","nsp8","nsp9","nsp10","helicase","exonuclease","endornase","S","E","M","N","ORF3a","ORF6","ORF7a","ORF8","RdRp","methyltransferase"]
  let methods = ["meme_full","fade","fel","relax","bgm","busted","meme","prime","cfel","absrel","slac"]
  let taskNames = genes.flatMap(d => methods.map(v => "selection_analyses_" + d + "." + v))


  onMount(async () => {

    let url = "http://silverback.temple.edu/veg/airflow/api/v1/dags/rascl_" + lineage + "/dagRuns"
    // let url = "http://silverback.temple.edu/veg/airflow/api/v1/dags/rascl_" + lineage + "/dagRuns/scheduled__2021-08-22T00:00:00+00:00/taskInstances?limit=500"
    let username = "datamonkey";
    let password = "0cc4m5R0ck5";
    let dagRunId = '';

    // Get dagRun id of latest completed run (state not running)

    // the `slug` parameter is available because
    // this file is called [slug].svelte
    let headers = new Headers();
    headers.set('Authorization', 'Basic ' + btoa(username + ":" + password));

    const res = await fetch(url, {
        method:'GET',
        headers: headers,
       });

    const data = await res.json();

    if (res.status === 200) {
      dagData = R.reject(d => d.state == 'running', data.dag_runs)
      dagRunId = R.last(dagData).dag_run_id;
      console.log(taskData);
    } else {
      console.log(res.status, data.message);
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
      console.log(taskData);
    } else {
      console.log(taskRes.status, taskRawData.message);
      return;
    }



  });

</script>

<style>
</style>

<svelte:head>
	<title>Lineage</title>
</svelte:head>

<h1>{lineage}</h1>

<div class="content">
  <table>
  {#each taskData as task}
    <tr>
      <td>{task.task_id}</td><td>{task.state}</td>
    </tr>
  {/each}
  </table>
</div>
