<template>
	<div class="config-editor">
		<template v-if="categories">
			<config-category v-for="category in categories" v-if="categoryFields(category.name).length" :key="category.name" :name="category.name">
				<component :is="componentFromField(field)" v-for="field in categoryFields(category.name)" :key="field.param" class="form-item--config" :schema="field" :current-value="model[field.paramName]" @update="updateModel"></component>
			</config-category>

			<config-category v-if="uncategorizedFields.length" key="Other" :name="$t('other')">
				<component :is="componentFromField(field)" v-for="field in uncategorizedFields" :key="field.param" class="form-item--config" :schema="field" :current-value="model[field.paramName]" @update="updateModel"></component>
			</config-category>
		</template>

		<template v-if="!categories">
			<fieldset class="config-uncategorized">
				<component :is="componentFromField(field)" v-for="field in uncategorizedFields" :key="field.param" class="form-item--config" :schema="field" :current-value="model[field.paramName]" @update="updateModel"></component>
			</fieldset>
		</template>
	</div>
</template>

<script>
	import InputString from './ConfigFields/InputString.vue';
	import InputBoolean from './ConfigFields/InputBoolean.vue';
	import InputNumber from './ConfigFields/InputNumber.vue';
	import InputFlag from './ConfigFields/InputFlag.vue';
	import InputSet from './ConfigFields/InputSet.vue';
	import InputList from './ConfigFields/InputList.vue';
	import InputTag from './ConfigFields/InputTag.vue';
	import InputEnum from './ConfigFields/InputEnum.vue';
	import InputDictionary from './ConfigFields/InputDictionary.vue';
	import InputUnknown from './ConfigFields/InputUnknown.vue';
	import ConfigCategory from './ConfigCategory.vue';

	export default {
		name: 'config-editor',
		components: { ConfigCategory },
		props: {
			fields: {
				type: Array,
				required: true
			},
			model: {
				type: Object,
				required: true
			},
			categories: Array,
			descriptions: Object,
			extendedFields: Object
		},
		computed: {
			uncategorizedFields() {
				if (!this.categories) return this.fields;

				const categorizedFields = this.categories.map(category => category.fields).reduce((categorizedFields, categoryFields) => [...categorizedFields, ...categoryFields], []);

				return this.fields.filter(field => !categorizedFields.includes(field.param));
			},
			categoryFields() {
				return categoryName => {
					if (!this.categories) return [];
					const category = this.categories.find(({ name }) => name === categoryName);
					if (!category) return [];
					return this.getFields(category.fields).sort((a, b) => category.fields.indexOf(a.paramName) - category.fields.indexOf(b.paramName));
				};
			},
			isValid() {
				return this.$children.every(child => child.isValid);
			}
		},
		mounted() {
			window.addEventListener('resize', this.computeLabelWidth);
			this.computeLabelWidth();
		},
		beforeDestroy() {
			window.removeEventListener('resize', this.computeLabelWidth);
		},
		methods: {
			componentFromField(field) {
				switch (field.type) {
					case 'string':
					case 'uint64':
						return InputString;
					case 'boolean':
						return InputBoolean;
					case 'uint32':
					case 'uint16':
					case 'byte':
						return InputNumber;
					case 'flag':
						return InputFlag;
					case 'enum':
						return InputEnum;
					case 'hashSet':
					case 'list':
						if (['enum'].includes(field.values.type)) return field.type === 'list' ? InputList : InputSet;
						if (['byte', 'uint16', 'uint32', 'uint64', 'string'].includes(field.values.type)) return InputTag;
						return InputUnknown;
					case 'dictionary':
						return InputDictionary;
					default:
						return InputUnknown;
				}
			},
			updateModel(value, field) {
				const fieldSchema = this.fields.find(fieldSchema => fieldSchema.paramName === field);

				if (fieldSchema && typeof fieldSchema.defaultValue !== 'undefined' && this.isDefault(value, fieldSchema)) {
					delete this.model[field];
				} else {
					this.model[field] = value;
				}
			},
			isDefault(value, fieldSchema) {
				return this.isEqual(value, fieldSchema.defaultValue, fieldSchema.type);
			},
			isEqual(a, b, type) {
				if (typeof a !== typeof b) return false;

				switch (type) {
					case 'uint32':
					case 'byte':
					case 'uint16':
					case 'uint64':
					case 'string':
					case 'boolean':
						return a === b;
					case 'hashSet':
						return a.length === b.length && a.every(item => b.includes(item));
					case 'list':
						return a.length === b.length && a.every((item, index) => item === b[index]);
					case 'dictionary':
						return Object.keys(a).length === Object.keys(b).length && Object.keys(a).every(key => a[key] === b[key]);
				}

				return false;
			},
			getFields(names) {
				return this.fields.filter(field => names.includes(field.param));
			},
			computeLabelWidth() {
				this.$el.style.setProperty('--label-width', 'auto');

				this.$nextTick(() => {
					const labelWidth = Math.max(...Array.from(this.$el.querySelectorAll('.form-item__label')).map(el => Math.ceil(parseFloat(getComputedStyle(el).width))));
					this.$el.style.setProperty('--label-width', `${labelWidth}px`);
				});
			}
		}
	};
</script>

<style lang="scss">
	.config-editor {
		margin-bottom: 1em;

		&:last-child {
			margin-bottom: 0;
		}
	}

	.config-uncategorized {
		border: 0 solid var(--color-border);
		padding: 0 1em 1em;
	}
</style>
