<template>
	<div class="vue-component styled-description" :class="{compact: compact}" v-html="markup(description)"></div>
</template>

<script>
import commonmark from 'commonmark';
import './base.css';

export default {
	name: 'Description',
	props: {
		description: String,
		preprocessor: Function,
		compact: {
			type: Boolean,
			default: false
		}
	},
	methods: {
		markup(text) {
			// Preprocess data
			if (typeof this.preprocessor === 'function') {
				text = this.preprocessor(text);
			}
			// Parse CommonMark
			var reader = new commonmark.Parser();
			var writer = new commonmark.HtmlRenderer();
			var parsed = reader.parse(text);
			return writer.render(parsed);
		}
	}
}
</script>

<style>
.vue-component.styled-description {
	line-height: 1.25em;
}
.vue-component.styled-description.compact p {
	margin: 0.5em 0;
}
.vue-component.styled-description.compact p:first-child {
	margin-top: 0;
}
.vue-component.styled-description.compact p:last-child {
	margin-bottom: 0;
}
.vue-component.styled-description pre {
	background-color: #eee;
	width: 100%;
	border: 1px solid #ccc;
	max-height: 15em;
	overflow-y: auto;
}
.vue-component.styled-description.compact pre {
	max-height: 7em;
	width: auto;
	max-width: 100%;
}
.vue-component.styled-description pre code {
	background-color: transparent; 
	display: block;
	margin: 0.5em;
}
.vue-component.styled-description code {
	color: maroon;
	display: inline-block;
	padding: 0 0.1em;
}
</style>
