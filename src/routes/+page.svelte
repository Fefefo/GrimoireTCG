<script lang="ts">
	import { Textarea } from "$lib/components/ui/textarea/index.js";
	import * as Button from "$lib/components/ui/button/index.js";
	import Text from "$lib/components/ui/text/text.svelte";

	type ygoproScheme = {
		data: {
			id: number;
			name: string;
		}[];
	};

	type scryfallCard = {
		name: string;
		scryfall_uri: string;
	};

	type scryfallScheme = {
		data: scryfallCard[];
	};

	let scryCards = $state<scryfallCard[]>([]);

	const apiYGOPRO = "https://db.ygoprodeck.com/api/v7/cardinfo.php?id=";

	let input = $state("");
	let output = $state("");

	let sum = $state(0);
	let stringAfterNumber = "x";

	let regex = /^(\d+)?(\s+)?([^(\n\r]*)(\(.*)?/;

	async function elaborate() {
		scryCards = [];
		sum = 0;
		output = input
			.split(/\r?\n/)
			.map((line) => {
				if (line.trim() == "") return "";
				if (line.startsWith("#") || line.startsWith("!")) return line;
				line = line.trim().replace(regex, (_, n = 1, s, text) => {
					const texttrim = text.trim();
					scryCards.push({ name: texttrim, scryfall_uri: "" });
					return `${n}${stringAfterNumber} ${texttrim}`;
				});
				sum += Number(line.match(/^(\d+)/)![0]);

				return line;
			})
			.join("\n");
		const cardsList = await fetchScryfall(scryCards.map((el) => el.name));
		console.log(cardsList);
	}

	async function fetchScryfall(cardsName: string[]): Promise<scryfallCard[]> {
		let cards: scryfallCard[] = [];
		const uniqueCards = [...new Set(cardsName)];
		for (let i = 0; i < uniqueCards.length; i += 75) {
			const cardsChunk = uniqueCards.slice(i, (i += 75));
			console.log(cardsChunk);
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
			console.log(response);
			const schema: scryfallScheme = await response.json();
			console.log(schema);
			schema.data.forEach((el) => {
				cards.push({ name: el.name, scryfall_uri: el.scryfall_uri.split("?")[0] });
			});
		}
		return cards;
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
</script>

<div class="flex min-h-screen w-full flex-col">
	<div class="flex w-full flex-1 flex-row flex-nowrap">
		<div class="flex w-1/5 items-stretch bg-black opacity-20"></div>
		<div class="w-3/5">
			<Text as="h1" class="pt-6 pb-8 text-center">Deck Formatter</Text>
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
					<Textarea
						bind:value={output}
						class="w-full resize-none"
						readonly
						placeholder="Your formatted list..."
					></Textarea>
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
