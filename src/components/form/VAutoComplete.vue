<template>
    <div ref="container" class="dropdown">
        <label
            v-if="$slots.label"
            :for="id"
            class="form-label"
        >
            <slot name="label"></slot>
        </label>

        <div :class="inputClassNames">
            <input
                v-model="value"
                type="text"
                :id="id"
                :disabled="disabled"
                :placeholder="placeholder"
                class="form-reset flex-fill"
                autocomplete="off"
                @input="onInput(value)"
                @focus="onFocus"
            >

            <button
                v-if="clearable && !isEmpty(modelValue)"
                type="button"
                class="btn-close"
                @click="clearItem"
            />
        </div>

        <ul :class="menuClassNames">
            <template v-if="isLoading">
                <li class="dropdown-item-text">
                    {{ $t('Loading') }}...
                </li>
            </template>

            <template v-else-if="items.length === 0">
                <li class="dropdown-item-text">
                    {{ $t('No data available') }}
                </li>
            </template>

            <template v-else-if="filteredItems.length === 0">
                <li class="dropdown-item-text">
                    {{ $t('No data found') }}
                </li>
            </template>

            <template v-else>
                <li
                    v-for="(item, index) of filteredItems"
                    :key="index"
                    :class="itemClassNames(item)"
                    @click="selectItem(item)"
                >
                    <slot
                        name="item"
                        :item="item.raw"
                        :text="item.text"
                        :value="item.value"
                    >
                        {{ item.text }}
                    </slot>
                </li>

                <VLazyLoadHelper tag="li" @reach="filterItems(search)"/>
            </template>
        </ul>

        <div
            v-if="errors.length !== 0"
            class="invalid-feedback"
        >
            {{ errors[0] }}
        </div>
    </div>
</template>

<script>
    // Vue
    import { ref, watch, computed } from 'vue';

    // Utils
    import { getUniqueId, isEmpty, resolveIteratee } from '@/utils';

    // Composables
    import { onClickOutside } from '@/composables/click.composable';
    import { useRegisterFormValidator } from '@/composables/validatation.composable';

    // Enums
    import ComponentSize from '@/enums/ComponentSize';

    // Components
    import VLazyLoadHelper from '@/components/VLazyLoadHelper.vue';

    export default {
        name: 'VAutoComplete',

        components: {
            VLazyLoadHelper
        },

        props: {
            modelValue: {
                required: true
            },
            items: {
                type: Array,
                required: true
            },
            itemKey: {
                type: [Function, String],
                default: 'key'
            },
            itemText: {
                type: [Function, String],
                default: 'text'
            },
            id: {
                type: String,
                default: () => getUniqueId()
            },
            size: {
                type: String,
                default: ComponentSize.MD,
                validator(value) {
                    return [
                        ComponentSize.XS,
                        ComponentSize.SM,
                        ComponentSize.MD,
                        ComponentSize.LG,
                        ComponentSize.XL
                    ].includes(value);
                }
            },
            placeholder: {
                type: String,
                default: null
            },
            clearable: {
                type: Boolean,
                default: false
            },
            disabled: {
                type: Boolean,
                default: false
            },
            isLoading: {
                type: Boolean,
                default: false
            },
            rules: {
                type: Array,
                default: () => []
            }
        },

        emits: ['update:modelValue'],

        setup(props, { emit }) {
            const { errors, isValid, validate } = useRegisterFormValidator();

            const isShown = ref(false);

            const container = ref();

            const inputClassNames = computed(() => {
                return [
                    'd-flex',
                    'form-control',
                    `form-control-${props.size}`,

                    {
                        'is-invalid': !isValid.value
                    }
                ];
            });

            const menuClassNames = computed(() => {
                return [
                    'dropdown-menu',
                    'autocomplete-menu',

                    {
                        'show': isShown.value
                    }
                ];
            });

            const itemClassNames = computed(() => {
                return (item) => [
                    'dropdown-item',

                    {
                        'active': item.value === props.modelValue
                    }
                ];
            });

            const resolveItemKey = resolveIteratee(props.itemKey);
            const resolveItemText = resolveIteratee(props.itemText);

            const COUNT_PER_PAGE = 8;

            const value = ref('');
            const search = ref('');

            const filteredItems = ref([]);

            let lastIndex = -1;

            function filterItems(value = '') {
                let count = 0;
                let index;

                value = value.toLowerCase();

                for (index = lastIndex + 1; index < props.items.length; index++) {
                    const item = props.items[index];
                    const text = String(resolveItemText(item));

                    if (value && !text.toLowerCase().includes(value)) {
                        continue;
                    }

                    filteredItems.value.push({
                        raw: item,
                        value: resolveItemKey(item),
                        text: text
                    });

                    count++;

                    if (count === COUNT_PER_PAGE) {
                        break;
                    }
                }

                lastIndex = index;
            }

            function resetFilterItems(value = '') {
                lastIndex = -1;
                filteredItems.value = [];

                filterItems(value);
            }

            let selectedItem;

            function clearItem() {
                selectedItem = undefined;
                emit('update:modelValue', undefined);

                value.value = '';
                search.value = '';

                resetFilterItems();
            }

            function selectItem(item) {
                selectedItem = item;
                emit('update:modelValue', item.value);

                hideHandler();
            }

            let offHandler;

            function hideHandler() {
                value.value = selectedItem !== undefined ? selectedItem.text : '';
                search.value = '';

                isShown.value = false;

                validate();
                resetFilterItems();

                offHandler();
            }

            function onInput(value) {
                search.value = value;
                resetFilterItems(value);
            }

            function onFocus() {
                isShown.value = true;

                const { off } = onClickOutside(container, hideHandler);
                offHandler = off;
            }

            function sync() {
                if (
                    selectedItem !== undefined
                    && selectedItem.value === props.modelValue
                ) {
                    return;
                }

                const selectedRawItem = props.items.find((item) => {
                    return resolveItemKey(item) === props.modelValue;
                });

                if (selectedRawItem === undefined) {
                    return;
                }

                selectedItem = {
                    raw: selectedRawItem,
                    value: resolveItemKey(selectedRawItem),
                    text: resolveItemText(selectedRawItem)
                };

                value.value = selectedItem.text;
            }

            watch([
                () => props.modelValue,
                () => props.items
            ], () => {
                resetFilterItems();
                sync();
            }, {
                immediate: true
            });

            return {
                errors,

                isShown,

                container,
                inputClassNames,
                menuClassNames,
                itemClassNames,

                value,
                search,
                filteredItems,
                filterItems,
                clearItem,
                selectItem,

                onInput,
                onFocus,

                isEmpty
            };
        }
    };
</script>