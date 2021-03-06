<template>
	<div class="vue-component json-schema" v-if="showSchema">
		<template v-if="visible">
			<div v-if="isProcessGraph" class="schemaProcessGraph">
				<div class="process-graph-parameters">
					<template v-if="Array.isArray(schema.parameters) && schema.parameters.length > 0">
						<strong>The following parameters are passed to the process:</strong>
						<ProcessParameter v-for="(param, i) in schema.parameters" :key="i" :parameter="param" :processReferenceParser="processReferenceParser" />
					</template>
					<strong v-else>No parameters are passed to the process.</strong>
				</div>
			</div>
			<div v-else-if="showRow('object')" class="schemaObjectElement">
				<div v-if="filteredObjectSchema !== null" class="inline-schema-attrs">
					<JsonSchema :schema="filteredObjectSchema" :nestingLevel="nestingLevel+1" />
				</div>
				<table class="object-properties">
					<tr>
						<th colspan="2" class="object-prop-heading">Object Properties:</th>
					</tr>
					<tr v-for="(val, key) in schema.properties" :key="key">
						<td class="propKey">
							{{ key }}
							<strong class="required" v-if="schema.required && schema.required.indexOf(key) !== -1" title="required">*</strong>
						</td>
						<td class="value">
							<JsonSchema :schema="val" :nestingLevel="nestingLevel+1" :processReferenceParser="processReferenceParser" />
						</td>
					</tr>
				</table>
			</div>
			<table v-else class="schema-attrs">
				<tr v-if="typeof schema.title == 'string'">
					<td colspan="2"><strong>{{ schema.title }}</strong></td>
				</tr>
				<tr v-if="typeof schema.description == 'string'">
					<td colspan="2"><Description :description="schema.description" :compact="true" /></td>
				</tr>
				<tr v-if="showAnyType">
					<td class="key">{{ formatKey('type') }}:</td>
					<td class="value data-type">any</td>
				</tr>
				<template v-else-if="isCompositeType">
					<tr>
						<th colspan="2" class="data-types-heading">Data Types:</th>
					</tr>
					<tr>
						<td colspan="2" class="schema-container data-types-container">
							<JsonSchema v-for="(v, k) in compositeTypes" :key="k" :schema="v" :nestingLevel="nestingLevel+1" :processReferenceParser="processReferenceParser" />
						</td>
					</tr>
				</template>
				<template v-if="!Array.isArray(this.schema)">
					<tr v-for="(val, key) in schema" :key="key">
						<template v-if="showRow(key)">
							<td class="key">{{ formatKey(key) }}:</td>
							<td class="value">
								<span v-if="key == 'type'" class="data-type">{{ formatType() }}</span>
								<div v-else-if="key == 'allOf' && Array.isArray(val)" class="schema-container">
									<JsonSchema v-for="(v, k) in val" :key="k" :schema="v" :nestingLevel="nestingLevel+1" :processReferenceParser="processReferenceParser" />
								</div>
								<span v-else-if="key != 'default' && key != 'examples' && val === true" title="true">✓ Yes</span>
								<span v-else-if="key != 'default' && key != 'examples' && val === false" title="false">✕ No</span>
								<ul v-else-if="key != 'examples' && Array.isArray(val)" class="comma-separated-list">
									<li v-for="(v, k) in val" :key="k">{{ v }}</li>
								</ul>
								<ul v-else-if="key == 'examples' && Array.isArray(val) && val.length > 1">
									<li v-for="(v, k) in val" :key="k"><code>{{ v }}</code></li>
								</ul>
								<code v-else-if="key == 'examples' && Array.isArray(val) && val.length === 1">{{ val[0] }}</code>
								<Description v-else-if="key == 'description'" :description="val" :compact="true" />
								<em v-else-if="key == 'default' && val === ''">Empty string</em>
								<code v-else-if="key == 'default' && (typeof val === 'object' || typeof val === 'boolean')">{{ JSON.stringify(val) }}</code>
								<code v-else-if="key == 'pattern'">{{ val }}</code>
								<JsonSchema v-else-if="typeof val === 'object'" :schema="val" :initShown="nestingLevel < 3" :nestingLevel="nestingLevel+1" :processReferenceParser="processReferenceParser" />
								<span v-else>{{ val }}</span>
							</td>
						</template>
					</tr>
				</template>
			</table>
		</template>
		<div class="schema-expand" v-else><a @click="show()">> ...</a></div>
	</div>
</template>

<script>
import Description from './Description.vue';
import Utils from '../utils.js';
import { CommonUtils } from '@openeo/js-commons';
import './base.css';

export default {
	name: 'JsonSchema',
	props: {
		schema: Object | Array,
		initShown: {
			type: Boolean,
			default: true
		},
		nestingLevel: {
			type: Number,
			default: 1
		},
		processReferenceParser: Function
	},
	data() {
		return {
			visible: this.initShown,
			filteredObjectSchema: null
		};
	},
	components: {
		Description
	},
	beforeCreate() {
		// See https://vuejs.org/v2/guide/components-edge-cases.html#Circular-References-Between-Components
		this.$options.components.ProcessParameter = require('./ProcessParameter.vue').default
	},
	created() {
        this.updateData();
	},
	computed: {
		showSchema() {
			return typeof this.schema === 'object' && this.schema !== null && this.nestingLevel < 20;
		},
		showAnyType() {
			return Utils.isAnyType(this.schema);
		},
		isProcessGraph() {
			return (this.schema.type === 'object' && this.schema.subtype === 'process-graph');
		},
		isCompositeType() {
			return (Array.isArray(this.schema) || Array.isArray(this.schema.anyOf) || Array.isArray(this.schema.oneOf));
		},
		compositeTypes() {
			if (Array.isArray(this.schema)) {
				return this.schema;
			}
			else if (Array.isArray(this.schema.anyOf)) {
				return this.schema.anyOf;
			}
			else if (Array.isArray(this.schema.oneOf)) {
				return this.schema.oneOf;
			}
			return [this.schema];
		}
	},
	watch: {
		initShown(newVal, oldVal) {
			this.visible = newVal;
		},
		schema() {
			this.updateData();
		}
	},
	methods: {
		updateData() {
			var filtered = null;
			for(var key in this.schema) {
				if (key == 'required' || key == 'properties' || key == 'parameters') {
					continue;
				}
				if (filtered === null) {
					filtered = {};
				}
				filtered[key] = this.schema[key];
			}
			this.filteredObjectSchema = filtered;
		},
		show() {
			this.visible = true;
		},
		formatKey(key) {
			switch(key) {
				case 'items':
					key = 'Array items';
					break;
				case 'minItems':
					key = 'Min. number of items';
					break;
				case 'const':
					key = 'Constant value';
					break;
				case 'maxItems':
					key = 'Max. number of items';
					break;
				case 'minimum':
					key = 'Minimum value (inclusive)';
					break;
				case 'maximum':
					key = 'Maximum value (inclusive)';
					break;
				case 'exclusiveMinimum':
					key = 'Minimum value (exclusive)';
					break;
				case 'exclusiveMinimum':
					key = 'Maximum value (exclusive)';
					break;
				case 'enum':
					key = 'Allowed values';
					break;
				case 'default':
					key = 'Default value';
					break;
				case 'type':
					key = 'Data type';
					break;
				case 'allOf':
					key = 'Composite data type';
					break;
				case 'contentMediaType':
					key = 'Media Type';
					break;
				case 'contentEncoding':
					key = 'Encoding';
					break;
				case 'deprecated':
					key = 'Deprecated';
					break;
				default:
					if (key.length > 1) {
						key = key.charAt(0).toUpperCase() + key.slice(1);
					}
			}
			return key;
		},
		formatType(schema) {
			if (typeof schema === 'undefined') {
				schema = this.schema;
			}
			return Utils.dataType(schema);
		},
		showRow(key) {
			if (key == 'object') {
				return (this.schema.type == 'object' && typeof this.schema.properties == 'object');
			}
			else if (key == 'title' || key == 'description' || key == 'subtype' || key == 'format' || key == 'anyOf' || key == 'oneOf') {
				return false;
			}
			else if (key == 'items' && Object.keys(this.schema.items).length === 1 && typeof this.schema.items.type !== 'undefined') {
				// If items hold only the type (is added to type anyway)
				return false;
			}

			return true;
		}
	}
}
</script>

<style>
.vue-component .schemaProcessGraph h4 {
	font-size: 1.1em;
	margin-top: 1em;
}
</style>

<style scoped>
.json-schema {
	border-left: 7px solid #ccc;
	border-bottom: 1px dotted #ccc;
	padding: 0.25%;
	width: 99%;
}
.data-types-container > .json-schema {
	border-left: 1px solid #ccc;
	border-bottom: 1px dotted #ccc;
	margin-top: 0.5em;
}
.json-schema td, .schemaProcessGraph {
	padding: 0.25em;
}
.inline-schema-attrs .json-schema {
	border: 0;
}
.schema-name {
	display: inline-block;
	border-bottom: 1px dotted black;
}
.schema-attrs {
	width: 100%;
}
.schema-attrs .key {
	min-width: 8em;
	width: 18%;
}
.schema-attrs .value {
	width: 82%;
}

.inline-schema-attrs .json-schema {
	background-color: transparent;
}
.object-prop-heading, .data-types-heading {
	padding: 0.5em 0em;
	text-align: left;
}
.object-properties .propKey {
	font-style: italic;
	font-weight: bold;
	min-width: 80px;
	width: 8%;
}
.object-properties th {
	padding-top: 1em;
}
</style>