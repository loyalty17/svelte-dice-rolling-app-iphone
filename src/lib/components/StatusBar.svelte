<script>
	import { Battery, Wifi, Signals } from './icons';
	$: currentTime = () => {
		let date = new Date();
		return `${(date.getHours() < 10 ? '0' : '') + date.getHours()}:${
			(date.getMinutes() < 10 ? '0' : '') + date.getMinutes()
		}`;
	};
</script>

<div
	class="absolute top-1 flex items-center justify-between w-full px-4 border-b border-slate-200 pb-2"
>
	<div class="text-xs font-bold pt-1">
		{currentTime()}
	</div>
	<div class="flex items-end justify-center gap-1">
		<Signals />
		<Wifi />
		<Battery />
	</div>
</div>
