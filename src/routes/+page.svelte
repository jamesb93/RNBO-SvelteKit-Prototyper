<script>
	import { Tooltip, DataTable, FileUploader, FileUploaderDropContainer, Button, Slider, Select, SelectItem, TextInput, Accordion, AccordionItem } from "carbon-components-svelte";
	import { onMount } from 'svelte';
	import RNBO from '@rnbo/js';
	
	let ctx;
	let device;
	let output;
	let buffers = [];
	let rnboConsole = {}

	function loadDevice(audioContext, patchJSON) {
		return new Promise((resolve, reject) => {
			RNBO.createDevice({ context: audioContext, patcher: patchJSON })
				.then(device => {
					if (device) {
						resolve(device);
					} else {
						reject('Error');
					}
				});
    	});
	}

	function initAudio() {
		ctx = new (window.AudioContext || window.webkitAudioContext)();
		output = ctx.createGain().connect(ctx.destination);
	}

	onMount(async() => {});
</script>

<svelte:head>
<link
rel="stylesheet"
href="https://unpkg.com/carbon-components-svelte/css/g10.css"
/>
</svelte:head>

<div class="loading">
	{#if !ctx}
	<Button 
	disabled={ctx}
	on:click={initAudio}
	>
	{ ctx ? 'Audio Started' : 'Start Audio' }
	</Button>
	{/if}

	{#if ctx}
	<FileUploaderDropContainer
	width="100%"
	labelText="Load a JSON patch by drag/drop or clicking here..."
	on:change={e => {
		let reader	 = new FileReader();
		reader.onload = read => {
			let json = JSON.parse(read.target.result);
			buffers = json.desc.externalDataRefs

			loadDevice(ctx, json)
			.then(x => {
				device = x;
				device.node.connect(output);
				device.messageEvent.subscribe(e => {
					if (e.tag !== 'valout') {
						rnboConsole[e.tag] = e.payload;
					}
				})
			})
		}
		reader.readAsText(e.detail[0]);
	}}
	/>
	{/if}
</div>
{#if device}
<Accordion size='xl'>
<AccordionItem open>
	<svelte:fragment slot="title">
		<h2>Parameters, Inports and Buffers</h2>
	  </svelte:fragment>
<div class="input-area">
	<div class="parameters">
		<div>Parameters</div>
		{#each device.parameters.sort((a, b) => a.type > b.type) as param}
			{#if param.type === 0}
			<Slider 
			fullWidth
			labelText={param.id}
			min={param.min} 
			max={param.max} 
			bind:value={param.value} 
			name={param.id}
			step={ param.steps > 1 ? (param.max-param.min) / (param.steps-1) : (param.max-param.min) / 1000.0 }
			/>
			{:else if param.type === 5}
			<Select
			labelText={param.id}
			on:change={e => param.value = e.target.selectedIndex}
			>
				{#each param.G as option}
				<SelectItem value={option} />
				{/each}
			</Select>
			{/if}
		{/each}
	</div>
	<div class="messages">
		<div class='title'>
			Inports
			<Tooltip>
				<p>Only numeric inputs are recognised and must be space separated</p>
			</Tooltip>
		</div>
		{#each device.messages.filter(message => message.type === RNBO.MessagePortType.Inport) as message}
		<TextInput size='sm' labelText={message.tag} placeholder=''
		on:change={e => {
			const raw = e.detail;
			const arr = raw.split(/\s+/).map(x => parseFloat(x)).filter(x => !isNaN(x));
			if (arr.length) {
				const messageEvent = new RNBO.MessageEvent(RNBO.TimeNow, message.tag, arr);
				device.scheduleEvent(messageEvent);
			}
		}}
		/>
		{/each}
	</div>
	<div class="buffers">
		<div class='title'>
			Buffers
			<Tooltip>
				<p>Supply audio files to buffers that are external in the patch.</p>
			</Tooltip>
		</div>
		{#each buffers as b}
		<!-- <FileUploaderDropContainer 
		width='100%'
		labelText={`Load a sample for ${b.tag} named ${b.id}`}
		on:change={e => {
			let reader = new FileReader();
			reader.onload = async read => {
				let decodedData = await ctx.decodeAudioData(read.target.result);
				await device.setDataBuffer(b.id, decodedData);
			}
			reader.readAsArrayBuffer(e.detail[0]);
		}}
		/> -->
		<FileUploader 
		labelDescription={`Load a sample for ${b.tag} named ${b.id}`}
		accept={['.aif', '.aiff', '.wav', '.mp3']}
		buttonLabel='Load samples'
		on:change={e => {
			let reader = new FileReader();
			reader.onload = async read => {
				let decodedData = await ctx.decodeAudioData(read.target.result);
				await device.setDataBuffer(b.id, decodedData);
			}
			reader.readAsArrayBuffer(e.detail[0]);
		}}
		status='complete'
		/>
		{/each}
	</div>
</div>
</AccordionItem>
</Accordion>
{/if}


<div class="outputs">
	{#if device}
	<DataTable 
	title='Outports and Print'
	description='Reports all outport and print object data'
	headers={[
		{ key: 'name', value: 'Name', width: '20%' },
		{ key: 'data', value: 'Data', width: '80%'}
	]}
	rows={Object.entries(rnboConsole).map(([k, v]) => ({
		id: k,
		name: k,
		data: v
	}))}
	/>
	{/if}
</div>

<style>
	.title {
		display: flex;
		flex-direction: row;
	}
	.input-area {
		display: flex;
		flex-direction: row;
		justify-content: space-between;
		gap: 3em;
		min-width: 1200px
	}
	.parameters, .messages, .buffers {
		display: flex;
		flex-direction: column;
		height: 100%;
		gap: 0.5em;
		width: 100%
	}

	@media screen and (max-width: 1200px) {
		.input-area {
			flex-direction: column;
			min-width: 100%;
		}
		.parameters, .messages, .buffers {
			width: 100%
		}
	}
</style>
