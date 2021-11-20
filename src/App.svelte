<script lang="ts">
	import Peer from 'peerjs';
	import nacl from 'tweetnacl';
	import base64 from '@stablelib/base64';

	var peer = new Peer();
	let peerid = ""
	let connection = null;
	let connected = false;
    export let currentMessage: string = "";

	let keypair = nacl.box.keyPair();
	let targetPublicKeyBytes;
	$: currentMessageBytes = new TextEncoder().encode(currentMessage);
	let messages = [];

	peer.on('open', function(id) {
		peerid = id;
	});

	peer.on('connection', function(conn) {
		connection = conn;
		connection.on('open', function() {
			connected = true;
			connection.send(
				JSON.stringify({
					type: "keyEx",
					publicKey: keypair.publicKey
				})	
			)
		})
		connection.on('data', function(data) {
			let message = JSON.parse(data);
			switch (message.type) {
				case "keyEx":
					targetPublicKeyBytes = Uint8Array.from(Object.values(message.publicKey));
					messages  = [...messages, {sender: "system", value: `Now chatting with ${base64.encode(targetPublicKeyBytes)}`}]
					break;
				case "blob":
					let nonce = Uint8Array.from(Object.values(message.nonce));
					let msg = nacl.box.open(Uint8Array.from(Object.values(message.msg)), nonce, targetPublicKeyBytes, keypair.secretKey);
					messages  = [...messages, {sender: "other", value: new TextDecoder().decode(msg)}]
					break;
			}

		});
	});


	function handleSend() {
		var nonce = nacl.randomBytes(nacl.box.nonceLength)
		let encryptedMessage = nacl.box(currentMessageBytes, nonce, targetPublicKeyBytes, keypair.secretKey)
		connection.send(JSON.stringify({
			type: "blob",
			nonce: nonce,
			msg: encryptedMessage
		}))
		messages = [...messages, {sender: "self", value: currentMessage}]
		currentMessage = ""
	}

	function handleConnection() {
		var conn = peer.connect(currentMessage);
		connection = conn;
		conn.on('open', function() {
			connected = true

			connection.on('data', function(data) {
				let message = JSON.parse(data);
				switch (message.type) {
				case "keyEx":
					targetPublicKeyBytes = Uint8Array.from(Object.values(message.publicKey));
					messages  = [...messages, {sender: "system", value: `Now chatting with ${base64.encode(targetPublicKeyBytes)}`}]				
					break;
				case "blob":
					let nonce = Uint8Array.from(Object.values(message.nonce));
					let msg = nacl.box.open(Uint8Array.from(Object.values(message.msg)), nonce, targetPublicKeyBytes, keypair.secretKey);
					messages  = [...messages, {sender: "other", value: new TextDecoder().decode(msg)}]
					break;
				}
			});
			connection.send(
				JSON.stringify({
					type: "keyEx",
					publicKey: keypair.publicKey
				})	
			)
		})
		currentMessage = ""
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
		{#if connected}
		<div class="w-100">
			<div class="pa-2 w-100">
				{#each messages as msg}
					<div class="w-100 flex pv1 ph2">
						{#if msg.sender == "self"}
							<div class="w-60 fl"></div>
							<div class="w-40 measure-wide br-pill br2-ns pa3 bg-green white">{msg.value}</div>
						{:else if msg.sender == "system"}
							<div class="w-100 center">{msg.value}</div>
						{:else}
							<div class="w-40 measure-wide br-pill br2-ns pa3 bg-black-10">{msg.value}</div>
							<div class="w-60"></div>
						{/if}
					</div>
				{/each}
			</div>
		</div>
		{:else}
		<button class="f6 link dim ba ph3 pv2 mb2 dib near-black" on:click={() => {
			navigator.clipboard.writeText(peerid)
		}}>Your Peer ID is {peerid} 📋</button>
		{/if}
		<div class="w-100">
			{#if connected}
				<div class="flex">
					<input class="fl bn w-90 pa-2 input-reset black-80 bg-white pa3 lh-solid w-100 w-75-m w-80-l br2-ns br--left-ns" bind:value={currentMessage} />
					<button class="fl w-10 pa-2 button-rese pv3 tc bn bg-animate bg-black-70 hover-bg-black white pointer w-100 w-25-m w-20-l br2-ns br--right-ns" on:click={() => handleSend()}>Send</button>
				</div>
			{:else}
				<div class="flex">
					<input class="fl bn w-90 pa-2 input-reset black-80 bg-white pa3 lh-solid w-100 w-75-m w-80-l br2-ns br--left-ns" bind:value={currentMessage} placeholder="Target ID"/>
					<button class="fl w-10 pa-2 button-rese pv3 tc bn bg-animate bg-blue hover-bg-black white pointer w-100 w-25-m w-20-l br2-ns br--right-ns" on:click={() => handleConnection()}>Connect</button>
				</div>
			{/if}
		</div>
	</div>
</main>

<style>
</style>