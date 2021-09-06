<script>
  import { onMount } from "svelte";
	import * as R from "ramda";

  let clades = [];
	let cladeFilter = '';
	let filteredClades = R.filter((d) => d.includes(cladeFilter, d), clades);

  onMount(async () => {
    const res = await fetch("http://data.hyphy.org/web/covid-19/voc.json");
    const data = await res.json()
    clades = data['clades'].sort();
		filteredClades = R.filter((d) => d.includes(cladeFilter, d), clades);
  });

	function handleKeydown(event) {
		filteredClades = R.filter((d) => d.includes(cladeFilter, d), clades);	
	}

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

	.voc-container {
		width: 100%;
		display: flex;
		flex-wrap: wrap;
	}


</style>

<svelte:head>
	<title>RASCL</title>
</svelte:head>

<h2>SARS-CoV-2 Analyses by Datamonkey</h2>

<h1>Sliding Windows</h1>


<h1>RASCL</h1>

<div class="grid grid-flow-col grid-cols-1 grid-auto-rows gap-1">
	<button class="rounded p-4 m-4 bg-blue-100 ring-blue-50 ring-8 ring-inset">Summary</button>
</div>

<div class="p-8">
  <div class="flex items-center rounded-full">
    <input bind:value={cladeFilter} on:keydown={handleKeydown} class="w-full py-4 px-6 text-gray-700 leading-tight focus:outline-none" id="search" type="text" placeholder="Search">
	</div>
</div>

<div class="voc-container grid grid-flow-col grid-cols-3 grid-auto-rows gap-1">
{#each filteredClades as clade}
  <a href="/lineage/{clade}" class="rounded p-4 m-4 w-60 bg-blue-100 ring-blue-50 ring-8 ring-inset">{clade}</a>
{/each}
</div>

