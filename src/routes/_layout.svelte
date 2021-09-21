<script>
	import Nav from '../components/Nav.svelte';

	export let segment;
</script>

<style>
  .bg {
    background-color: rgba(0,0,0,0.1);
    background-image: url(https://github.com/veg/datamonkey-js/raw/master/public/assets/img/repeat-bg_360.jpg);
    background-repeat: repeat;
  }

  .bg2 {
    background-color: rgba(255,255,255,.9);
  }

	main {
		position: relative;
		max-width: 100em;
		padding: 2em;
		margin: 0 auto;
		box-sizing: border-box;
	}

</style>

<Nav {segment} />

<div class="bg">
  <div class="bg2">
    <main>
      <slot></slot>
    </main>
  </div>
</div>
