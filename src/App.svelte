<script lang="ts">
	import Peer from 'peerjs';
	var peer = new Peer();
	let peerid = ""
	let connection = null;
	let connected = false;
	let idx = 0;
	let currentMessageBytes;
    export let currentMessage: string = "";

	let fileInput;
	let oneTimePad = [
		"dldfjasd;flakjsdf;laksjdfaa;lkjwe;lrkqjwe;lrkjq;l4k34l5kj34;lkj345;l23k4j;l3kj3452345iheofiedflkdhgsdfgkj;slerkj;lw3k4j5;l34kj52;3l4kjdfnsdfksd;rlktwje4;l3kj45;l23k4j53;l4kjerkjwerltkjwerkj34523k4j543252j3k4wefwerlkwj432l34k5j2345234lkjerwer3434345kejsdfg4",
		"dldfjasd;flakjsdf;laksjdfaa;lkjwe;lrkqjwe;erkjq;l4k34l5kj34;lkj345;l23k4j;l3kj3452345iheofiedflkdhgsdfgkj;slerkj;lw3k4j5;l34kj52;3l4kjdfnsdfksd;rlktwje4;l3kj45;l23k4j53;l4kjerkjwerltkjwerkj34523k4j543252j3k4wefwerlkwj432l34k5j2345234lkjerwer3434345kejsdfg4",
		"dldfjasd;flakjsdf;laksjdfaa;lkjwe;lrkqjwe;lrkjq;l4k34lwkj34;lkj345;l23k4j;l3kj3452345iheofiedflkdhgsdfgkj;slerkj;lw3k4j5;l34kj52;3l4kjdfnsdfksd;rlktwje4;l3kj45;l23k4j53;l4kjerkjwerltkjwerkj34523k4j543252j3k4wefwerlkwj432l34k5j2345234lkjerwer3434345kejsdfg4",
		"dldfjasd;flakjsdf;laksjdfaa;lkjwe;lrkqjwe;lrkjq;l4kw4l5kj34;lkj345;l23k4j;l3kj3452345iheofiedflkdhgsdfgkj;slerkj;lw3k4j5;l34kj52;3l4kjdfnsdfksd;rlktwje4;l3kj45;l23k4j53;l4kjerkjwerltkjwerkj34523k4j543252j3k4wefwerlkwj432l34k5j2345234lkjerwer3434345kejsdfg4",
		"dldfjasd;flakjsdf;laksjdfaa;lkjwe;ldkqjwe;lrkjq;l4k34l5kj34;lkj345;l23k4j;l3kj3452345iheofiedflkdhgsdfgkj;slerkj;lw3k4j5;l34kj52;3l4kjdfnsdfksd;rlktwje4;l3kj45;l23k4j53;l4kjerkjwerltkjwerkj34523k4j543252j3k4wefwerlkwj432l34k5j2345234lkjerwer3434345kejsdfg4",
		"dldfjasd;flakjsdf;laksjdfaa;lkjwe;lrkqjwe;lrkjq;l4k34l5kj34;lkj345;l2wk4j;l3kj3452345iheofiedflkdhgsdfgkj;slerkj;lw3k4j5;l34kj52;3l4kjdfnsdfksd;rlktwje4;l3kj45;l23k4j53;l4kjerkjwerltkjwerkj34523k4j543252j3k4wefwerlkwj432l34k5j2345234lkjerwer3434345kejsdfg4",
	];
	let messages = [];

	peer.on('open', function(id) {
  		console.log('My peer ID is: ' + id);
		peerid = id;
	});

	peer.on('connection', function(conn) {
		connection = conn;
		connection.on('open', function() {
			connected = true;
		})
		connection.on('data', function(data) {
	    	console.log('Received', data);
			messages  = [...messages, {sender: "other", value: data}]
		});
	});

	$: currentMessageBytes = new TextEncoder().encode(currentMessage);
	$: currentPeerId = peerid;

	function encryptMessage(msg, idx) {
		let padBytes = new TextEncoder().encode(oneTimePad[idx]);
		let encodedMessage = [];
		for (var i = 0; i < padBytes.length; i++) {
			if (i < currentMessageBytes.length) {
				encodedMessage.push(padBytes[i] ^ currentMessageBytes[i]);
			} else {
				encodedMessage.push(padBytes[i] ^ 32)
			}
		}
		return encodedMessage
	}

	function handleSend() {
		console.log(currentMessage)
		let encryptedMessage = encryptMessage(currentMessage, idx)
		console.log(encryptedMessage)
		connection.send(currentMessage)
		messages = [...messages, {sender: "self", value: currentMessage}]
		currentMessage = ""
		idx = idx+1;
	}

	function handleConnection() {
		var conn = peer.connect(currentMessage);
		connection = conn;
		conn.on('open', function() {
			connected = true
			connection.on('data', function(data) {
	    		console.log('Received', data);
				messages  = [...messages, {sender: "other", value: data}]
			});
		})
		currentMessage = ""
	}

	const onFileSelected = (e)=>{
  		let file = e.target.files[0];
        let reader = new FileReader();
        reader.readAsText(file);
        reader.onload = e => { oneTimePad = JSON.parse(e.target.result) };
	}

</script>

<svelte:window on:keydown={event => {
		if (event.key == "Enter") {
			if (connected) {
				handleSend()
			} else {
				handleConnection()
			}
		}
	}}/>

<main class="w-100 bg-black-10 sans-serif ph2">
	<div class="flex flex-column justify-between vh-100">
		<div class="w-100">
			<div class="pa-2 w-100">
				{#each messages as msg}
					<div class="w-100 flex pv1 ph2">
						{#if msg.sender == "self"}
							<div class="w-60 fl"></div>
							<div class="w-40 measure-wide br-pill br2-ns pa3 bg-green white">{msg.value}</div>
						{:else}
							<div class="w-40 measure-wide br-pill br2-ns pa3 bg-black-10">{msg.value}</div>
							<div class="w-60"></div>
						{/if}
					</div>
				{/each}
			</div>
		</div>
		<div class="w-100">
			<!--
			<input type="file" accept=".json" on:change={(e)=>onFileSelected(e)} bind:this={fileInput} >
			-->
			
			{#if connected}
				<p>{256 - currentMessageBytes.length} bytes remaining, {oneTimePad.length - idx} keys remaining</p>
				<div class="flex">
					<input class="fl bn w-90 pa-2 input-reset black-80 bg-white pa3 lh-solid w-100 w-75-m w-80-l br2-ns br--left-ns" bind:value={currentMessage} />
					<button class="fl w-10 pa-2 button-rese pv3 tc bn bg-animate bg-black-70 hover-bg-black white pointer w-100 w-25-m w-20-l br2-ns br--right-ns" on:click={() => handleSend()}>Send</button>
				</div>
			{:else}
				<p>Your ID is {currentPeerId}</p>
				<div class="flex">
					<input class="fl bn w-90 pa-2 input-reset black-80 bg-white pa3 lh-solid w-100 w-75-m w-80-l br2-ns br--left-ns" bind:value={currentMessage} placeholder="Enter ID to connect to"/>
					<button class="fl w-10 pa-2 button-rese pv3 tc bn bg-animate bg-blue hover-bg-black white pointer w-100 w-25-m w-20-l br2-ns br--right-ns" on:click={() => handleConnection()}>Connect</button>
				</div>
			{/if}
		</div>
	</div>
</main>

<style>
</style>