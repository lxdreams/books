<template>
    <el-table-column v-bind="bindData">
        <template v-if="column.children && column.children.length">
            <edit-table-column
                v-for="item of column.children"
                :key="item.property"
                :column="item"
                :overflowTip="overflowTip"
                :enableEdit="enableEdit"
                :enableValidation="enableValidation"/>
        </template>
        <template slot-scope="scope">
            <template v-if="enableEdit">
                <slot v-if="column.slot" :name="column.slot" :data="scope.row"></slot>
                <el-input
                    v-else-if="!column.type || column.type ==='text'"
                    type="textarea"
                    v-model="scope.row[scope.column.property]"
                    @blur="validate(scope)"
                    autosize
                    :readonly="readOnly(scope)"
                    :clearable="column.clearable"/>
                <el-input
                    type="number"
                    v-else-if="column.type === 'number'"
                    v-model="scope.row[scope.column.property]"
                    @blur="validate(scope)"
                    :readonly="readOnly(scope)"
                    :clearable="column.clearable"/>
                <el-date-picker
                    v-else-if="['date', 'datetime'].includes(column.type)"
                    :type="column.type"
                    v-model="scope.row[scope.column.property]"
                    :format="column.format"
                    :value-format="column.valueFormat || column.type === 'date'?'yyyy-MM-dd':'yyyy-MM-dd HH:mm:ss'"
                    @change="validate(scope)"
                    :editable="false"
                    prefix-icon="-"
                    :readonly="readOnly(scope)"
                    :disabled="readOnly(scope)"
                    :clearable="column.clearable"/>
                <el-time-picker
                    v-else-if="column.type==='time'"
                    v-model="scope.row[scope.column.property]"
                    :format="column.format"
                    :value-format="column.valueFormat || 'HH:mm:ss'"
                    @change="validate(scope)"
                    :readonly="readOnly(scope)"
                    :disabled="readOnly(scope)"
                    :clearable="column.clearable">
                </el-time-picker>
                <el-select
                    v-else-if="column.type === 'select'"
                    v-model="scope.row[scope.column.property]"
                    @change="validate(scope)"
                    :disabled="readOnly(scope)"
                    :multiple="column.multiple"
                    :options="column.options"
                    :filterable="column.filterable"
                    :clearable="column.clearable">
                    <el-option
                        v-for="(itm,index) of (column.options || asyncOptions)"
                        :key="index"
                        :label="itm.label"
                        :value="itm.value">
                    </el-option>
                </el-select>
                <tree-select
                    v-else-if="column.type === 'treeSelect'"
                    v-model="scope.row[scope.column.property]"
                    @change="validate(scope)"
                    :disabled="readOnly(scope)"
                    :multiple="column.multiple"
                    :options="column.options || asyncOptions"
                    :filterable="column.filterable"
                    :clearable="column.clearable"/>
                <el-checkbox
                    v-else-if="item.type === 'checkbox'"
                    v-model="scope.row[scope.column.property]"
                    @blur="validate(scope)"
                    :disabled="readOnly(scope)">
                </el-checkbox>
                <div class="is-error" v-show="errorMap[`${column.property}-${scope.$index}`]">
                    {{errorMap[`${column.property}-${scope.$index}`]}}
                </div>
            </template>
            <template v-else-if="column.slot">
                <slot :name="column.slot" :data="scope.row"></slot>
            </template>
            <span v-else-if="column.formatter">
              {{column.formatter(scope.row[scope.column.property], scope.row)}}
            </span>
            <span v-else v-html="scope.row[scope.column.property]">
            </span>
        </template>
    </el-table-column>
</template>

<script>
    import {initData} from '@/api/data'
    import TreeSelect from '@/components/tree-select/index.vue'

    export default {
        name: "TableColumn",
        components: {
            TreeSelect
        },
        props: {
            column: {
                type: Object,
                required: true,
            },
            overflowTip: {
                type: Boolean,
                default: true,
            },
            enableEdit: {
                type: Boolean,
                default: false,
            },
            enableValidation: {
                type: Boolean,
                default: true,
            }
        },
        inject: ['editTable'],
        computed: {
            bindData() {
                const obj = {
                    label: this.column.label,
                    className: this.column.className,
                    labelClassName: this.column.labelClassName,
                    property: this.column.prop || this.column.property,
                    width: this.column.width,
                    minWidth: this.column.minWidth,
                    id: this.column.columnId,
                    columnKey: this.column.columnKey || this.column.prop || this.column.property,
                    sortable: this.column.sortable === '' ? true : this.column.sortable,
                    sortMethod: this.column.sortMethod,
                    sortBy: this.column.sortBy,
                    resizable: this.column.resizable,
                    showOverflowTooltip: this.overflowTip || this.column.showOverflowTooltip || this.column.showTooltipWhenOverflow,
                    align: this.column.align || 'left',
                    formatter: this.column.formatter,
                    selectable: this.column.selectable,
                    reserveSelection: this.column.reserveSelection,
                    fixed: this.column.fixed === '' ? true : this.column.fixed,
                    filterMethod: this.column.filterMethod,
                    filters: this.column.filters,
                    filterable: (this.column.filters && this.column.filters.length) || this.column.filterMethod,
                    filterMultiple: this.column.filterMultiple,
                    filteredValue: this.column.filteredValue || [],
                    filterPlacement: this.column.filterPlacement || '',
                    index: this.column.index,
                    sortOrders: this.column.sortOrders,
                    key: Math.random(),
                }
                for (const key of Object.keys(obj)) {
                    if (!obj[key] || obj[key].length === 0) {
                        delete obj[key]
                    }
                }
                return obj
            },
        },
        data() {
            return {
                errorMap: {},
                asyncOptions: [],
            }
        },
        methods: {
             /**
             * 规则验证
             * @param scope
             */
            validate(scope) {
                const readonly = this.readOnly(scope);
                let validateMsg = '';
                if (this.enableEdit && this.enableValidation && !readonly) {
                    const rules = this.column.rules;
                    const val = scope.row[scope.column.property];
                    if (rules && rules.hasOwnProperty('required') && rules.required && !val) {
                        validateMsg = rules.message
                    } else if (rules && rules.hasOwnProperty('validator')) {
                        const fun = rules.validator;
                        const res = fun(val);
                        if (res) {
                            validateMsg = res.msg;
                            if (res.clear) scope.row[scope.column.property] = ''
                        }
                    }
                }
                if (validateMsg) {
                    this.$set(this.errorMap, `${this.column.property}-${scope.$index}`, validateMsg)
                } else {
                    this.$delete(this.errorMap, `${this.column.property}-${scope.$index}`)
                }
            },

            /**
             * 仅可读
             * @param scope
             * @returns {*|boolean}
             */
            readOnly(scope) {
                return this.column.readonly || !scope.row.enableEdit
            },

            /**
             * 根据url获取option
             */
            getAsyncOptions() {
                const {optionsUrl, params} = this.column
                if (optionsUrl) {
                    initData(optionsUrl, params).then(res => {
                        if (res.code === 200) {
                            this.asyncOptions = res.data;
                        }
                    }).catch(err => {
                        this.$message.error(err.message);
                    });
                }
            },
        }
    }
</script>

<style scoped>
    .is-error {
        color: #ff4949;
        font-size: 12px;
        line-height: 1;
    }
</style>
