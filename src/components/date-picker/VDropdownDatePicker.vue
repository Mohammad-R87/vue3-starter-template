<template>
    <div class="dropdown-date-picker">
        <label v-if="$slots.label" class="form-label">
            <slot name="label"></slot>
        </label>

        <div class="dropdown-date-picker-container">
            <div class="col-3">
                <VAutoComplete
                    v-model="selected.day"
                    :items="days"
                    :placeholder="$t('Day')"
                    class="dropdown-date-picker-day"
                />
            </div>

            <div class="col-6">
                <VAutoComplete
                    v-model="selected.month"
                    :items="months"
                    :placeholder="$t('Month')"
                    class="dropdown-date-picker-month"
                />
            </div>

            <div class="col-3">
                <VAutoComplete
                    v-model="selected.year"
                    :items="years"
                    :placeholder="$t('Year')"
                    class="dropdown-date-picker-year"
                />
            </div>
        </div>

        <div
            v-if="errors.length !== 0"
            class="invalid-feedback d-block"
        >
            {{ errors[0] }}
        </div>
    </div>
</template>

<script>
    // Vue
    import { ref, watch, reactive, computed } from 'vue';

    // Utils
    import {
        hasNullObject
    } from '@/utils';
    import DateTime from '@/utils/date-time';

    // Composables
    import { useRegisterFormValidator } from '@/composables/validatation.composable';

    // Constants
    import DateTimeConstants from '@/constants/date-time';

    // Components
    import VAutoComplete from '@/components/form/VAutoComplete.vue';

    export default {
        name: 'VDropdownDatePicker',

        components: {
            VAutoComplete
        },

        props: {
            timestamp: {
                type: Number,
                required: true
            },
            start: {
                type: Number,
                default: DateTime.today().subYear(100).getTimestamp()
            },
            end: {
                type: Number,
                default: DateTime.today().getTimestamp()
            },
            calendar: {
                type: String,
                default: DateTimeConstants.Jalali,
                validator(value) {
                    return [
                        DateTimeConstants.Gregorian,
                        DateTimeConstants.Jalali,
                        DateTimeConstants.Islamic
                    ].includes(value);
                }
            },
            rules: {
                type: Array,
                default: () => []
            }
        },

        emits: ['update:timestamp'],

        setup(props, { emit }) {
            const { errors, validate } = useRegisterFormValidator(true, 'timestamp');

            function initializePickerDate() {
                if (!props.timestamp) {
                    return DateTime.today().setCalendar(props.calendar);
                }

                return DateTime.make(props.timestamp).setCalendar(props.calendar);
            }

            /** @type {Ref<DateTime>} **/
            const pickerDate = ref(initializePickerDate());

            /** @type {Ref<DateTime>} **/
            const startDate = ref(DateTime.make(props.start));

            /** @type {Ref<DateTime>} **/
            const endDate = ref(DateTime.make(props.end));

            function initializeSelectedDate() {
                if (!props.timestamp) {
                    return {
                        day: null,
                        month: null,
                        year: null
                    };
                }

                return {
                    day: pickerDate.value.clone().getDay(),
                    month: pickerDate.value.clone().getMonth(),
                    year: pickerDate.value.clone().getYear()
                };
            }

            const selected = reactive(initializeSelectedDate());

            const days = computed(() => {
                const items = [];

                let daysInMonth = 31;

                for (let index = 1; index <= daysInMonth; index++) {
                    items.push({
                        key: index,
                        text: index
                    });
                }

                return items;
            });

            const months = computed(() => {
                const items = [];

                const date = DateTime.today().setCalendar(props.calendar).startOfYear();

                for (let index = 1; index <= DateTimeConstants.MonthsPerYear; index++) {
                    items.push({
                        key: index,
                        text: date.format('MMMM')
                    });

                    date.addMonth(1);
                }

                return items;
            });

            const years = computed(() => {
                const items = [];

                const startYear = startDate.value.clone().setCalendar(props.calendar).getYear();
                const endYear = endDate.value.clone().setCalendar(props.calendar).getYear();

                for (let index = startYear; index <= endYear; index++) {
                    items.push({
                        key: index,
                        text: index
                    });
                }

                return items;
            });

            watch(selected, () => {
                if (hasNullObject(selected)) {
                    return;
                }

                pickerDate.value.setDay(selected.day);
                pickerDate.value.setMonth(selected.month);
                pickerDate.value.setYear(selected.year);

                emit('update:timestamp', pickerDate.value.clone().getTimestamp());

                validate();
            });

            return {
                selected,

                errors,

                days,
                months,
                years
            };
        }
    };
</script>