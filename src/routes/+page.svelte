<script lang="ts">
	import { Textarea } from "$lib/components/ui/textarea/index.js";
	import * as Button from "$lib/components/ui/button/index.js";
	import Text from "$lib/components/ui/text/text.svelte";
	import { SvelteMap } from "svelte/reactivity";
	import * as Tooltip from "$lib/components/ui/tooltip/index.js";
	import Copy from "$lib/assets/icons/copy.svelte";
	import Error from "$lib/assets/icons/error.svelte";

	type ygoproScheme = {
		data: {
			id: number;
			name: string;
		}[];
	};

	type scryfallCard = {
		name: string;
		count: number;
		scryfall_uri?: string;
		scryfall_image_uri?: string;
		scryfall_image_uri_back?: string;
	};

	type scryfallScheme = {
		data: {
			name: string;
			scryfall_uri: string;
			image_uris?: {
				border_crop: string;
			};
			card_faces?: {
				image_uris?: {
					border_crop: string;
				};
			}[];
		}[];
		not_found: {
			name: string;
		}[];
	};

	let cardCounter = new SvelteMap<string, [number, string]>();

	let scryCards = $state<scryfallCard[]>([]);

	const apiYGOPRO = "https://db.ygoprodeck.com/api/v7/cardinfo.php?id=";

	let input = $state("");

	let sum = $state(0);
	let stringAfterNumber = "x";

	let regex = /^(\d+)?(\s+)?([^(\n\r]*)(\(.*)?/;
	// Adrix and Nev twincasters
	async function elaborate() {
		scryCards = [];
		cardCounter = new SvelteMap<string, [number, string]>();
		sum = 0;
		let ordered: string[] = [];
		input.split(/\r?\n/).forEach((line) => {
			if (line.trim() == "") return "";
			if (line.startsWith("#") || line.startsWith("!")) return line;
			const matching = line.trim().match(regex);
			const count = parseInt(matching ? (matching[1] != undefined ? matching[1] : "1") : "1");
			if (matching && matching[3]) {
				const name = normalizeString(matching[3].split("//")[0].trim());
				if (cardCounter.has(name)) {
					cardCounter.set(name, [cardCounter.get(name)![0] + count, cardCounter.get(name)![1]]);
				} else {
					cardCounter.set(name, [count, matching[3]]);
				}
				// CardCounter.set(name, (CardCounter.get(name) ?? 0) + 1);
				ordered.push(matching[3].trim());
				sum += count;
			}
		});
		scryCards = await fetchScryfall(Array.from(cardCounter.keys()));
		scryCards.map((el) => {
			if (!el.scryfall_image_uri) {
				el.name = ordered.find((elArr) => normalizeString(elArr) == el.name) ?? el.name;
			}
		});
		scryCards = sortByFirstOccurrence(ordered, scryCards);
	}

	async function fetchScryfall(cardsName: string[]): Promise<scryfallCard[]> {
		scryCards = [];
		for (let i = 0; i < cardsName.length; i += 75) {
			const cardsChunk = cardsName.slice(i, (i += 75));
			const response = await fetch("https://api.scryfall.com/cards/collection", {
				method: "POST",
				headers: {
					"Content-Type": "application/json"
				},
				body: JSON.stringify({
					identifiers: cardsChunk.map((el) => {
						return { name: el };
					})
				})
			});
			const schema: scryfallScheme = await response.json();
			schema.data.forEach((el) => {
				if (el.name.split("//").length > 1) {
					el.name.split("//").forEach((e) => {
						if (cardCounter.has(normalizeString(e))) {
							cardCounter.set(normalizeString(el.name), cardCounter.get(normalizeString(e))!);
							cardCounter.delete(normalizeString(e));
						}
					});
				}
				if (!el.image_uris && el.card_faces) {
					scryCards.push({
						name: el.name,
						scryfall_uri: el.scryfall_uri.split("?")[0],
						count: cardCounter.get(normalizeString(el.name))![0],
						scryfall_image_uri: el.card_faces[0].image_uris?.border_crop,
						scryfall_image_uri_back: el.card_faces[1].image_uris?.border_crop
					});
				} else {
					scryCards.push({
						name: el.name,
						scryfall_uri: el.scryfall_uri.split("?")[0],
						count: cardCounter.get(normalizeString(el.name))![0],
						scryfall_image_uri: el.image_uris?.border_crop
					});
				}
			});
			schema.not_found.forEach((el) => {
				scryCards.push({ name: el.name, count: cardCounter.get(normalizeString(el.name))![0] });
			});
		}
		console.log(scryCards.length);
		console.log(cardsName.length);
		return scryCards;
	}

	async function ygoproparser() {
		let cards = input
			.split(/\r?\n/)
			.map((line) => {
				if (line.startsWith("#") || line.startsWith("!")) return "";
				return line.trim();
			})
			.filter((card) => card != "")
			.join(",");
		let result = await fetch(apiYGOPRO + cards);
		let json: ygoproScheme = await result.json();
		json.data.forEach((el) => {
			input = input.replaceAll(String(el.id), el.name);
		});
	}

	function normalizeString(str: string): string {
		return str
			.toLowerCase()
			.normalize("NFD")
			.replace(/[\u0300-\u036f]/g, "")
			.replace(/ß/g, "ss")
			.replace(/æ/g, "ae")
			.replace(/œ/g, "oe")
			.replace(/ø/g, "o")
			.replace(/[^\p{L}\p{N}]/gu, "");
	}

	function sortByFirstOccurrence(rawList: string[], scryCards: scryfallCard[]): scryfallCard[] {
		// 1. costruisco mappa: normalized -> primo indice
		const firstIndex = new SvelteMap<string, number>();

		rawList.forEach((str, i) => {
			const norm = normalizeString(str);

			// salva solo la prima occorrenza
			if (!firstIndex.has(norm)) {
				firstIndex.set(norm, i);
			}
		});

		// 2. ordino gli item
		return scryCards.sort((a, b) => {
			const indexA = firstIndex.get(normalizeString(a.name)) ?? Infinity;
			const indexB = firstIndex.get(normalizeString(b.name)) ?? Infinity;

			return indexA - indexB;
		});
	}
</script>

<div class="flex min-h-screen w-full flex-col">
	<div class="flex w-full flex-1 flex-row flex-nowrap">
		<div class="flex w-1/5 items-stretch bg-black opacity-20"></div>
		<div class="w-3/5">
			<Text as="h1" class="pt-6 pb-8 text-center">GrimoireTCG</Text>
			<div class="flex w-full flex-row flex-wrap justify-evenly">
				<div class="w-1/3">
					<Textarea
						bind:value={input}
						class="w-full resize-none"
						placeholder="Paste your list here."
					></Textarea>
					<p class="py-4">Number of cards: <span class="font-bold">{sum}</span></p>
				</div>
				<div class="w-1/3">
					<div
						class="mb-4 flex w-full resize-none flex-col justify-start border-gray-300 p-2"
						class:border={scryCards.length > 0}
					>
						{#each scryCards as card (card)}
							<Tooltip.Provider>
								<Tooltip.Root disableHoverableContent delayDuration={200}>
									<Tooltip.Trigger class="text-left">
										{#if card.scryfall_uri}
											<a href={card.scryfall_uri} rel="external" target="_blank">
												<span class="text-left">{card.count}{stringAfterNumber} {card.name}</span>
											</a>
										{:else}
											<span class="flex flex-row gap-2 text-left"
												>{card.count}{stringAfterNumber}
												{card.name}
												<div class="pt-1 text-yellow-200">
													<Error></Error>
												</div>
											</span>
										{/if}
									</Tooltip.Trigger>
									{#if card.scryfall_image_uri}
										<Tooltip.Content class="flex gap-2">
											<img src={card.scryfall_image_uri} alt="" class="h-40" />
											{#if card.scryfall_image_uri_back}
												<img src={card.scryfall_image_uri_back} alt="" class="h-40" />
											{/if}
										</Tooltip.Content>
									{/if}
								</Tooltip.Root>
							</Tooltip.Provider>
						{/each}
					</div>
					{#if scryCards.length != 0}
						<Button.Root
							class="bg-indigo-300"
							onclick={async () => {
								await navigator.clipboard.writeText(
									scryCards
										.map((el) => {
											return el.count + stringAfterNumber + " " + el.name;
										})
										.join("\n")
								);
							}}><Copy></Copy>Copy List</Button.Root
						>
					{/if}
				</div>
			</div>
			<div class="flex w-full justify-center gap-5 px-4 py-8">
				<Button.Root onclick={elaborate}>Format your decklist</Button.Root>
				<Button.Root onclick={ygoproparser} class="bg-indigo-400"
					>Convert IDs from YGOPRO</Button.Root
				>
			</div>
			<div class="flex justify-center">
				<div class="flex flex-col"></div>
			</div>
		</div>
		<div class="flex w-1/5 items-stretch bg-black opacity-20"></div>
	</div>
</div>
