<template>
	<v-drawer v-model="internalActive" :title="title" persistent @cancel="cancel">
		<template v-if="template !== null && templateData && primaryKey !== '+'" #title>
			<v-skeleton-loader v-if="loading || templateDataLoading" class="title-loader" type="text" />

			<h1 v-else class="type-title">
				<render-template :collection="templateCollection.collection" :item="templateData" :template="template" />
			</h1>
		</template>

		<template #subtitle>
			<v-breadcrumb :items="[{ name: collectionInfo.name, disabled: true }]" />
		</template>

		<template #actions>
			<slot name="actions" />
			<v-button v-tooltip.bottom="t('save')" icon rounded @click="save">
				<v-icon name="check" />
			</v-button>
		</template>

		<div class="drawer-item-content">
			<template v-if="junctionField">
				<file-preview
					v-if="file"
					:src="file.src"
					:mime="file.type"
					:width="file.width"
					:height="file.height"
					:title="file.title"
					:in-modal="true"
				/>

				<v-form
					:disabled="disabled"
					:loading="loading"
					:initial-values="item && item[junctionField]"
					:primary-key="relatedPrimaryKey"
					:model-value="internalEdits[junctionField]"
					:fields="junctionRelatedCollectionFields"
					:validation-errors="junctionField ? validationErrors : undefined"
					autofocus
					@update:model-value="setJunctionEdits"
				/>

				<v-divider v-if="showDivider" />
			</template>

			<v-form
				v-model="internalEdits"
				:disabled="disabled"
				:loading="loading"
				:initial-values="item"
				:primary-key="primaryKey"
				:fields="fields"
				:validation-errors="!junctionField ? validationErrors : undefined"
			/>
		</div>
	</v-drawer>
</template>

<script lang="ts">
import api from '@/api';
import FilePreview from '@/views/private/components/file-preview.vue';
import { set } from 'lodash';
import { computed, defineComponent, PropType, ref, toRefs, watch } from 'vue';
import { useI18n } from 'vue-i18n';

import { usePermissions } from '@/composables/use-permissions';
import { useTemplateData } from '@/composables/use-template-data';
import { useFieldsStore } from '@/stores/fields';
import { useRelationsStore } from '@/stores/relations';
import { unexpectedError } from '@/utils/unexpected-error';
import { validateItem } from '@/utils/validate-item';
import { useCollection } from '@directus/shared/composables';
import { Field, Relation } from '@directus/shared/types';

export default defineComponent({
	components: { FilePreview },
	props: {
		active: {
			type: Boolean,
			default: false,
		},
		collection: {
			type: String,
			required: true,
		},
		primaryKey: {
			type: [String, Number],
			default: null,
		},
		edits: {
			type: Object as PropType<Record<string, any>>,
			default: undefined,
		},
		junctionField: {
			type: String,
			default: null,
		},
		disabled: {
			type: Boolean,
			default: false,
		},
		// There's an interesting case where the main form can be a newly created item ('+'), while
		// it has a pre-selected related item it needs to alter. In that case, we have to fetch the
		// related data anyway.
		relatedPrimaryKey: {
			type: [String, Number],
			default: '+',
		},

		// If this drawer-item is opened from a relational interface, we need to force-block the field
		// that relates back to the parent item.
		circularField: {
			type: String,
			default: null,
		},
	},
	emits: ['update:active', 'input'],
	setup(props, { emit }) {
		const { t, te } = useI18n();

		const validationErrors = ref<any[]>([]);

		const fieldsStore = useFieldsStore();
		const relationsStore = useRelationsStore();

		const { internalActive } = useActiveState();
		const { junctionFieldInfo, junctionRelatedCollection, junctionRelatedCollectionInfo, setJunctionEdits } =
			useJunction();
		const { internalEdits, loading, item } = useItem();
		const { save, cancel } = useActions();

		const { collection } = toRefs(props);

		const { info: collectionInfo } = useCollection(collection);

		const isNew = computed(() => props.primaryKey === '+' && props.relatedPrimaryKey === '+');

		const title = computed(() => {
			const collection = junctionRelatedCollectionInfo?.value || collectionInfo.value!;

			if (te(`collection_names_singular.${collection.collection}`)) {
				return isNew.value
					? t('creating_unit', {
							unit: t(`collection_names_singular.${collection.collection}`),
					  })
					: t('editing_unit', {
							unit: t(`collection_names_singular.${collection.collection}`),
					  });
			}

			return isNew.value
				? t('creating_in', { collection: collection.name })
				: t('editing_in', { collection: collection.name });
		});

		const showDivider = computed(() => {
			return (
				fieldsStore.getFieldsForCollection(props.collection).filter((field: Field) => field.meta?.hidden !== true)
					.length > 0
			);
		});

		const { fields: junctionRelatedCollectionFields } = usePermissions(
			junctionRelatedCollection,
			computed(() => item.value && item.value[props.junctionField]),
			computed(() => props.primaryKey === '+')
		);

		const { fields: fieldsWithPermissions } = usePermissions(
			collection,
			item,
			computed(() => props.primaryKey === '+')
		);

		const fields = computed(() => {
			if (props.circularField) {
				return fieldsWithPermissions.value.map((field: Field) => {
					if (field.field === props.circularField) {
						set(field, 'meta.readonly', true);
					}
					return field;
				});
			} else {
				return fieldsWithPermissions.value;
			}
		});

		const fieldsWithoutCircular = computed(() => {
			if (props.circularField) {
				return fields.value.filter((field) => {
					return field.field !== props.circularField;
				});
			} else {
				return fields.value;
			}
		});

		const templatePrimaryKey = computed(() =>
			junctionFieldInfo.value ? String(props.relatedPrimaryKey) : String(props.primaryKey)
		);

		const templateCollection = computed(() => junctionRelatedCollectionInfo.value || collectionInfo.value);
		const { templateData, loading: templateDataLoading } = useTemplateData(templateCollection, templatePrimaryKey);

		const template = computed(
			() =>
				junctionRelatedCollectionInfo.value?.meta?.display_template ||
				collectionInfo.value?.meta?.display_template ||
				null
		);

		const { file, isDirectusFiles } = useFile();

		return {
			t,
			internalActive,
			internalEdits,
			loading,
			item,
			validationErrors,
			save,
			cancel,
			title,
			junctionFieldInfo,
			junctionRelatedCollection,
			setJunctionEdits,
			showDivider,
			junctionRelatedCollectionFields,
			fields,
			template,
			templateCollection,
			templatePrimaryKey,
			templateData,
			templateDataLoading,
			collectionInfo,
			file,
			isDirectusFiles,
		};

		function useFile() {
			const isDirectusFiles = computed(() => {
				return junctionRelatedCollection.value === 'directus_files';
			});

			const file = computed(() => {
				if (isDirectusFiles.value === false || !item.value) return null;
				const fileData = item.value?.[props.junctionField];
				if (!fileData) return null;

				const src = `assets/${fileData.id}?key=system-large-contain`;
				return { ...fileData, src };
			});

			return { file, isDirectusFiles };
		}

		function useActiveState() {
			const localActive = ref(false);

			const internalActive = computed({
				get() {
					return props.active === undefined ? localActive.value : props.active;
				},
				set(newActive: boolean) {
					localActive.value = newActive;
					emit('update:active', newActive);
				},
			});

			return { internalActive };
		}

		function useItem() {
			const localEdits = ref<Record<string, any>>({});

			const internalEdits = computed<Record<string, any>>({
				get() {
					return {
						...(props.edits ?? {}),
						...localEdits.value,
					};
				},
				set(newEdits) {
					localEdits.value = newEdits;
				},
			});

			const loading = ref(false);
			const item = ref<Record<string, any> | null>(null);

			watch(
				() => props.active,
				(isActive) => {
					if (isActive) {
						if (props.primaryKey !== '+') fetchItem();
						if (props.relatedPrimaryKey !== '+') fetchRelatedItem();
					} else {
						loading.value = false;
						item.value = null;
						localEdits.value = {};
					}
				},
				{ immediate: true }
			);

			return { internalEdits, loading, item, fetchItem };

			async function fetchItem() {
				loading.value = true;

				const endpoint = props.collection.startsWith('directus_')
					? `/${props.collection.substring(9)}/${props.primaryKey}`
					: `/items/${props.collection}/${encodeURIComponent(props.primaryKey)}`;

				let fields = '*';

				if (props.junctionField) {
					fields = `*,${props.junctionField}.*`;
				}

				try {
					const response = await api.get(endpoint, { params: { fields } });

					item.value = response.data.data;
				} catch (err: any) {
					unexpectedError(err);
				} finally {
					loading.value = false;
				}
			}

			async function fetchRelatedItem() {
				loading.value = true;

				const collection = junctionRelatedCollection.value;

				const endpoint = collection.startsWith('directus_')
					? `/${collection.substring(9)}/${props.relatedPrimaryKey}`
					: `/items/${collection}/${encodeURIComponent(props.relatedPrimaryKey)}`;

				try {
					const response = await api.get(endpoint);

					item.value = {
						...(item.value || {}),
						[junctionFieldInfo.value.field]: response.data.data,
					};
				} catch (err: any) {
					unexpectedError(err);
				} finally {
					loading.value = false;
				}
			}
		}

		function useJunction() {
			const junctionFieldInfo = computed(() => {
				if (!props.junctionField) return null;

				return fieldsStore.getField(props.collection, props.junctionField);
			});

			const junctionRelatedCollection = computed(() => {
				if (!props.junctionField) return null;

				// If this is a m2m/m2a, there will be 2 relations associated with this field
				const relations = relationsStore.getRelationsForField(props.collection, props.junctionField);

				const relationForField: Relation = relations.find((relation: Relation) => {
					return relation.collection === props.collection && relation.field === props.junctionField;
				});

				if (relationForField.related_collection) return relationForField.related_collection;
				if (relationForField.meta?.one_collection_field)
					return (
						props.edits[relationForField.meta.one_collection_field] ||
						item.value?.[relationForField.meta.one_collection_field]
					);
				return null;
			});

			const { info: junctionRelatedCollectionInfo } = useCollection(junctionRelatedCollection);

			return { junctionFieldInfo, junctionRelatedCollection, junctionRelatedCollectionInfo, setJunctionEdits };

			function setJunctionEdits(edits: any) {
				if (!props.junctionField) return;

				internalEdits.value = {
					...internalEdits.value,
					[props.junctionField]: edits,
				};
			}
		}

		function useActions() {
			return { save, cancel };

			function save() {
				const editsToValidate = props.junctionField ? internalEdits.value[props.junctionField] : internalEdits.value;
				const fieldsToValidate = props.junctionField
					? junctionRelatedCollectionFields.value
					: fieldsWithoutCircular.value;
				let errors = validateItem(editsToValidate || {}, fieldsToValidate, isNew.value);

				if (errors.length > 0) {
					validationErrors.value = errors;
					return;
				}

				emit('input', internalEdits.value);
				internalActive.value = false;
				internalEdits.value = {};
			}

			function cancel() {
				validationErrors.value = [];
				internalActive.value = false;
				internalEdits.value = {};
			}
		}
	},
});
</script>

<style lang="scss" scoped>
.v-divider {
	margin: 52px 0;
}

.drawer-item-content {
	padding: var(--content-padding);
	padding-bottom: var(--content-padding-bottom);
}
</style>
