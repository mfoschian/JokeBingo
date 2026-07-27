<script setup>
import { onMounted, ref, computed } from 'vue';


const forced_name = new URLSearchParams(location.search)?.get('d');


const DEFAULT_DATASET = 'default';
const DEFAULT_CARD = {
	title: null,
	sentences: [],
	guessed: []
};

const CARD_LENGTH = 5;

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
		card.value.sentences = randomChoose(dataset.sentences, CARD_LENGTH);
		card.value.guessed = [];
		card.value.title = dataset?.directives?.title ?? null;
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
	const yesno = confirm("Cambio la cartella (perderai le frasi indovinate...)?");
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
	<h1 class="title">{{ card.title ?? 'Joke' }} Bingo!</h1>
	<template v-if="status == STATUS.ERROR">
		<div class="error">
			<h2>Mi spiace, si è verificato un errore:</h2>
			<pre>
				{{ error }}
			</pre>
		</div>
	</template>
	<template v-if="status == STATUS.LOADING">
		<h2>Caricamento in corso</h2>
	</template>
	<template v-if="status == STATUS.WON">
		<h2>HAI VINTO !!!!</h2>
		<div class="sentences">
			<div class="sentence" v-for="(s,ix) in card.sentences">
				<input type="checkbox" disabled :checked="card.guessed[ix]" @input="setGuess(ix,$event.target.checked)">{{ s }}</input>
			</div>
		</div>
		<button @click="clearCardData()">Riprova con un'altra cartella</button>
	</template>
	<template v-if="status == STATUS.PLAYING">
		<button @click="changeCard()">Abbandona cartella</button>
		<hr>
		<div class="sentences">
			<div class="sentence" v-for="(s,ix) in card.sentences">
				<input type="checkbox" :checked="card.guessed[ix]" @input="setGuess(ix,$event.target.checked)">{{ s }}</input>
			</div>
		</div>
	</template>
</template>

<style scoped></style>
