<script setup>
import { onMounted, ref, computed } from 'vue';


const forced_name = new URLSearchParams(location.search)?.get('d');


const DEFAULT_DATASET = 'default';
const DEFAULT_CARD = {
	title: null,
	instructions: null,
	sentences: [],
	guessed: []
};

const DEFAULT_CARD_LENGTH = 5;

let last_dataset = ref(DEFAULT_DATASET);
let card = ref({});

const hasWon = computed( () => {
	let guessed = card.value.guessed.filter( g => g == true );
	return guessed.length > 0 && guessed.length == card.value.sentences.length;
});

const error = ref(null);

const STATUS = {
	LOADING: 1,
	PLAYING: 2,
	WON: 3,
	ERROR: 100
};

let status = ref(STATUS.LOADING);

function setError(err) {
	console.error(err);
	error.value = err;
	status.value = STATUS.ERROR;
}

async function fetchDataset(name) {
	const file = name ?? 'default';
	let dataset = {};
	try {
		let res = await fetch(`sentences/${file}.txt`);
		if( !res.ok )
			throw (res.status + ' ' +res.statusText);

		let txt = res.text();
		let lines = (await txt).split('\n').map(line => line.trim())
			.filter(line => line.length > 0 && !line.startsWith('#'));

		let directives = {}
		lines.filter( line => line.startsWith('@')).forEach( l => {
			const ix = l.indexOf(':');
			if(ix < 0) directives[key] = true;

			let key = l.slice(1,ix).trim().toLowerCase(); // skip '@'
			let value = l.slice(ix+1).trim();
			directives[key] = value;
		});
		const sentences = lines.filter( line => !line.startsWith('@'));
		dataset = { directives, sentences };
	}
	catch (err) {
		setError(err);
	}

	return dataset;
}

function randomChoose(dataset, num) {
	if(!Array.isArray(dataset) || isNaN(parseInt(num)) || num < 1)
		return [];

	const res = dataset.sort( (a,b) => Math.random() > 0.5 ? 1 : -1 ).slice(0, num);
	return res;
}

async function loadData() {

	status.value = STATUS.LOADING;

	// Get saved data from local storage
	let cardData = await readCardData();

	let dataset_name = cardData.name;
	let c = Object.assign({}, DEFAULT_CARD, cardData.card);
	console.log(c);

	if( !c.sentences || c.sentences.length == 0 ) {
		// Load sentences
		const dataset = await fetchDataset(dataset_name);
		card.value.sentences = randomChoose(dataset.sentences, dataset?.directives?.itemstowin ?? DEFAULT_CARD_LENGTH);
		card.value.guessed = [];
		card.value.title = dataset?.directives?.title ?? null;
		card.value.instructions = dataset?.directives?.instructions ?? null;
	}
	else {
		card.value = c;
	}

	// Save current card and dataset-name
	await writeCardData(dataset_name, card.value);
	last_dataset.value = dataset_name;

	if( status.value == STATUS.LOADING )
		status.value = hasWon.value ? STATUS.WON : STATUS.PLAYING;
}


async function readCardData() {
	try {
		// Get last used dataset name
		let dataset_name = forced_name ?? window.localStorage.getItem('last_dataset') ?? DEFAULT_DATASET;
		const text = window.localStorage.getItem('card_' + dataset_name );
		let c = null;
		if(text) {
			c = JSON.parse(text);
		}
		return { card: c, name: dataset_name };
	}
	catch( err ) {
		setError(err);
		return { error: err };
	}
}

async function writeCardData(name, data) {
	try {
		const dataset_name = forced_name ?? name ?? 'default';
		window.localStorage.setItem('last_dataset', dataset_name);
		window.localStorage.setItem('card_'+ dataset_name, JSON.stringify(data));
	}
	catch(err) {
		setError(err);
	}
}

async function changeCard() {
	const yesno = confirm("Se cambi cartella perderai le frasi indovinate. Sicuro ?");
	if(yesno == true)
		await clearCardData();
}

async function clearCardData() {
	try {
		let dataset_name = forced_name ?? window.localStorage.getItem('last_dataset') ?? DEFAULT_DATASET;
		window.localStorage.removeItem('card_' + dataset_name );
		window.localStorage.removeItem('last_dataset');
		await loadData();
	}
	catch( err ) {
		setError(err);		
	}
}

async function setGuess(ix, flag) {
	card.value.guessed[ix] = flag;
	await writeCardData(last_dataset.value, card.value);

	if(hasWon.value) {
		status.value = STATUS.WON;
	}
}


onMounted( loadData );
</script>

<template>
	<h1 >Joke Bingo!</h1>
	<template v-if="status == STATUS.ERROR">
		<div class="error">
			<h2>Mi spiace, si è verificato un errore:</h2>
			<pre>
				{{ error }}
			</pre>
		</div>
		<div class="buttons">
			<button @click="clearCardData()">Resetta e riprova</button>
		</div>
	</template>
	<template v-else-if="status == STATUS.LOADING">
		<h2>Caricamento in corso</h2>
	</template>
	<template v-else-if="status == STATUS.WON">
		<div class="title" v-if="card.title">{{ card.title }}</div>

		<div class="won">HAI VINTO !!!!</div>
		<div class="sentences">
			<div class="sentence" v-for="(s,ix) in card.sentences">
				<input type="checkbox" disabled :checked="card.guessed[ix]" @input="setGuess(ix,$event.target.checked)">{{ s }}</input>
			</div>
		</div>
		<div class="buttons">
			<button @click="clearCardData()">Gioca ancora con un'altra cartella</button>
		</div>
	</template>
	<template v-if="status == STATUS.PLAYING">
		<div class="instructions" v-if="card.instructions">{{ card.instructions }}</div>
		<div class="title" v-if="card.title">{{ card.title }}</div>

		<div class="sentences">
			<div class="sentence" v-for="(s,ix) in card.sentences" @click="setGuess(ix, ! (card.guessed[ix] ?? false))">
				<div class="checkbox">
					<input type="checkbox" :checked="card.guessed[ix]">{{ s }}</input>
				</div>
				<div class="thumb"><span>&#x1F44D;</span></div>
			</div>
		</div>
		<div class="buttons">
			<button @click="changeCard()">Cambia cartella</button>
		</div>
	</template>
</template>

<style>
#app {
	display: flex;
	flex-direction: column;
	align-items: center;
	gap: 1rem;

	& > * {
		max-width: 90%;
	}
}
body {
	background-color: #e7e7e7;
}
</style>

<style scoped lang="scss">

.title {
	font-weight: bold;
	font-size: 1.3rem;
	text-align: center;

	&:before,&:after {
		content: "\"";
	}
}

.won {
	z-index: 1000;
	padding: 3rem;
	font-size: 2rem;
	font-weight: bold;
	background-color: rgb(253, 64, 64);
	border-radius: 1rem;
	color: #ffe409;
	position: absolute;
	top: 30%;
	width: 45%;
	text-align: center;
	

	@keyframes tilt-n-move-shaking {
	  0% { transform: translate(0, 0) rotate(0deg); }
	  25% { transform: translate(5px, 5px) rotate(5deg); }
	  50% { transform: translate(0, 0) rotate(0eg); }
	  75% { transform: translate(-5px, 5px) rotate(-5deg); }
	  100% { transform: translate(0, 0) rotate(0deg); }
	}

	animation: tilt-n-move-shaking .3s infinite;
}

.instructions {
	border: 1px dashed black;
	border-radius: 1rem;
	padding: .8rem;

	font-style: italic;
	font-size: 1.1rem;
	text-align: center;
}

.sentences {
	display: flex;
	flex-direction: column;
	gap: 1rem;

	.sentence {
		--sentence-color: rgb(130, 253, 253);
		--sentence-color: white;
		--sentence-ok-color: #53e753;

		// justify-self: stretch;
		padding: 1rem;
		background-color: var(--sentence-color);
		border: 1px transparent contrast-color(var(--sentence-color));
		border-radius: .5rem;
		font-weight: bold;
		
		transition: background-color .5s ease-out;
		display: flex;
		flex-direction: row;
		flex-wrap: wrap;
		gap: 1rem;
		// justify-items: center;
		// justify-content: center;

		input[type="checkbox"] {
			appearance: none;
		}		

		overflow: hidden;
		.thumb {
			// display:none;
			flex-basis: 0;
			transform: rotate(-180deg);
			transition: transform .3s ease-in-out;
			
			span {
				padding-left:.5rem;
				padding-right:.5rem;
				opacity: 0;

				transition: opacity .3s ease-in-out;
			}
		}

		.checkbox {
			flex-basis: 1;
			flex-grow: 1;
			text-align: center;
		}

		&:has(input[type="checkbox"]:checked) {
			--sentence-color: var(--sentence-ok-color);

			.thumb {
				transform: rotate(0deg);

				span {
					opacity: 1;
				}
			}
		}		

	}
}


.buttons {
	margin-top: 1rem;

	button {
		--btn-color: beige;
		padding: .7rem;
		background-color: var(--btn-color);
		border: 1px solid contrast-color(var(--btn-color));
		border-radius: .7rem;

		box-shadow: 5px 5px 5px rgb(52, 51, 20);
	}
}
</style>
