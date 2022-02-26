<script lang="ts">
	import { onMount } from 'svelte';
	import { initialize, walletStore } from '$lib/walletStore';
	import {
		LedgerWalletAdapter,
		PhantomWalletAdapter,
		SlopeWalletAdapter,
		SolflareWalletAdapter,
		SolletExtensionWalletAdapter,
		SolletWalletAdapter,
		TorusWalletAdapter
	} from '@solana/wallet-adapter-wallets';
	import {
		WalletAdapterNetwork,
		WalletName,
		WalletNotConnectedError
	} from '@solana/wallet-adapter-base';
	import {
		Connection,
		Keypair,
		LAMPORTS_PER_SOL,
		SystemProgram,
		Transaction
	} from '@solana/web3.js';

	const localStorageKey = 'walletAdapter';
	// Change this to the desired Network Cluster
	const network: WalletAdapterNetwork = WalletAdapterNetwork.Devnet;

	const print = () => console.log($walletStore);

	async function selectWallet(walletName: string) {
		try {
			$walletStore.select(walletName as WalletName);
			$walletStore.connect();
		} catch (err) {
			console.error(err);
		}
	}

	function getRPCUrl() {
		switch (network) {
			case WalletAdapterNetwork.Devnet:
				return 'https://api.devnet.solana.com';
			case WalletAdapterNetwork.Testnet:
				return 'https://api.testnet.solana.com';
			case WalletAdapterNetwork.Mainnet:
				return 'https://api.mainnet-beta.solana.com';
		}
	}

	async function sendTransaction() {
		const publicKey = $walletStore.publicKey;
		const connection = new Connection(getRPCUrl());
		if (!publicKey) throw new WalletNotConnectedError();
		const transaction = new Transaction().add(
			SystemProgram.transfer({
				fromPubkey: publicKey,
				toPubkey: Keypair.generate().publicKey,
				lamports: 0.1 * LAMPORTS_PER_SOL
			})
		);
		const signature = await $walletStore.sendTransaction(transaction, connection);
		await connection.confirmTransaction(signature, 'recent');
	}

	onMount(async () => {
		// These will support the most popular wallets  but you can find the
		// full list of wallets here: https://github.com/solana-labs/wallet-adapter#wallets
		const adaptors = [
			new PhantomWalletAdapter(),
			new SlopeWalletAdapter(),
			new SolflareWalletAdapter({ network }),
			new TorusWalletAdapter(),
			new LedgerWalletAdapter(),
			new SolletWalletAdapter({ network }),
			new SolletExtensionWalletAdapter({ network })
		];
		initialize({
			wallets: adaptors,
			autoConnect: true,
			localStorageKey: localStorageKey,
			onError: (err) => console.error(err.message)
		});
	});
</script>

<div class="cols">
	{#if $walletStore?.connected}
		<div>
			{$walletStore.publicKey} is connected
		</div>
		<button on:click={() => $walletStore.disconnect()}>
			Disconnect {$walletStore.wallet.name}
			<img src={$walletStore.wallet.icon} alt={$walletStore.wallet.name} style="height: 20px" />
		</button>
		<br />
		<button on:click={sendTransaction}>Send Transaction</button>
	{:else}
		{#each $walletStore.wallets as wallet}
			<button class="btn" on:click={() => selectWallet(wallet.name)}>
				<img src={wallet.icon} alt={wallet.name} style="height: 20px" />
				{wallet.name}
			</button>
		{/each}
	{/if}
	<br />
	<button on:click={print}>Print wallet store state to console</button>
</div>

<style>
	.cols {
		padding: 22px;
		max-width: 300px;
		display: flex;
		flex-direction: column;
		justify-content: center;
	}
	.btn {
		margin: 10px;
	}
</style>
