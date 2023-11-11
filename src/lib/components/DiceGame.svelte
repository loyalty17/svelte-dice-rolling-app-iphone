<script lang="ts">
	import {
		StatusBar,
		Dice,
		Navigation
	} from '$lib/components/';
	
	let Game: any;
</script>

<div class="relative flex justify-center mt-2 w-[360px] mx-auto h-[710px]">
	<img src="/mockup.png" alt="" class="absolute mx-auto" />
	<div class="absolute top-6 left-0 flex justify-center w-full">
		<div class="rounded-[35px] overflow-hidden w-[320px] h-[687px] bg-transparent text-slate-700">
			<!-- svelte-ignore a11y-no-static-element-interactions -->
			<!-- svelte-ignore a11y-mouse-events-have-key-events -->
			<div id="Game" bind:this={Game} class="game">
				<StatusBar />
				<!-- put your game here -->
				<Dice />
				<Navigation />
			</div>
		</div>
	</div>
</div>

<style>
	.game {
		@apply flex justify-center items-center relative h-full;
	}
</style>
