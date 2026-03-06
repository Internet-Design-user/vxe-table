<template>
    <div class="search-box">
        <el-form inline :model="formData" :label-width="autoWidth" inline-message @validate="handleValidate" @submit.prevent>
            <div class="flex flex-warp">
                <slot />
                <el-form-item
                    v-for="(item, index) in formList"
                    :key="index"
                    :prop="item.prop"
                    :rules="item.rules"
                    :class="!item.isShowClose ? 'close-box' : 'pr20'"
                >
                    <template #label v-if="item.label">
                        <div class="flex-center pr12 shrink">
                            <span class="font12">{{ item.label }}</span>
                            <span v-if="item.tooltip">≈</span>
                        </div>
                    </template>
                    <div class="flex items-center">
                        <!-- 输入框类型 -->
                        <el-input
                            v-if="item.type === 'input'"
                            :ref="`${item.prop}Ref`"
                            v-model="formData[item.prop]"
                            :maxlength="item.maxlength"
                            clearable
                            :style="{ width: item.itemWidth || '200px' }"
                            :placeholder="item.placeholder || item.label"
                            @input="handleDateChange(item)"
                        />

                        <!-- 下拉选择器 -->
                        <el-select-v2
                            v-if="item.type === 'select' || item.type === 'selectGroup'"
                            :ref="`${item.prop}Ref`"
                            v-model="formData[item.prop]"
                            :style="{ width: item.itemWidth || '200px' }"
                            :options="item.list || []"
                            :disabled="item.disabled"
                            :clearable="item.clearable"
                            label-key="label"
                            value-key="value"
                            :multiple="item.multiple"
                            :multiple-limit="item.limit"
                            filterable
                            @change="handleCurrentChange(item)"
                        />
                        <el-date-picker
                            v-if="item.type === 'dateMonth'"
                            :ref="`${item.prop}Ref`"
                            v-model="formData[item.prop]"
                            type="month"
                            class="mr-[5px]"
                            value-format="YYYY-MM"
                            :placeholder="$t('page.pickMonth')"
                            @change="handleDateChange(item)"
                            :disabled-date="pickerOptionsForMonth"
                        />
                        <el-date-picker
                            v-if="item.type === 'dateDay'"
                            :ref="`${item.prop}Ref`"
                            class="mr-[5px]"
                            v-model="formData[item.prop]"
                            type="date"
                            value-format="YYYY-MM-DD"
                            :placeholder="$t('page.pickDay')"
                            :disabled-date="pickerOptions"
                            @change="handleDateChange(item)"
                        />
                        <!-- 日期选择器 -->
                        <el-date-picker
                            v-if="item.type === 'date'"
                            :ref="`${item.prop}Ref`"
                            v-model="formData[item.prop]"
                            type="datetimerange"
                            value-format="x"
                            :range-separator="$t('filter.to')"
                            :start-placeholder="$t('filter.startDate')"
                            :end-placeholder="$t('filter.endDate')"
                            @change="handleDateChange(item)"
                        />

                        <!-- 关闭按钮 -->
                        <el-link
                            type="info"
                            :class="['ml-[5px] close-btn', { 'visually-hidden': item.isShowClose }]"
                            :underline="false"
                            @click="handleCloseTarget(item)"
                            :icon="useRenderIcon(removeIcon)"
                        >
                        </el-link>
                    </div>

                    <!-- 错误提示 -->
                    <el-tooltip v-if="formErrors[item.prop]" popper-class="filter-custom-tooltip" placement="bottom" effect="light">
                        <template #content>
                            <p class="content">
                                <infoIcon class="icon-error mr-[5px]"></infoIcon>
                                {{ formErrors[item.prop] }}
                            </p>
                        </template>
                        <component :is="useRenderIcon(infoIcon)" :style="{ right: item.isShowClose ? '-15px' : '15px' }" />
                    </el-tooltip>
                </el-form-item>
                <!-- 过滤按钮区域 -->
                <div v-if="filterOptsList.length" class="filter-pos">
                    <el-button v-if="isFilter" type="primary" plain :icon="useRenderIcon(filterIcon)" @click="onSubmit('selectRef')">
                        <span>{{ $t('filter.filterTxt') }}</span>
                    </el-button>
                    <el-select
                        v-else
                        ref="selectRef"
                        class="filter-select"
                        v-model="selectKey"
                        @change="handleSelectChange"
                        @visible-change="handleVisible"
                    >
                        <template v-for="item in filterOptsList">
                            <el-option v-if="!item.isShowClose" :key="item.prop" :label="item.label" :value="item.prop" />
                        </template>
                    </el-select>
                </div>

                <!-- 清空按钮 -->
                <div v-if="isShowClearBtn" class="flex-center clear-btn" @click="handleClear">
                    <el-button :icon="useRenderIcon(removeIcon)" type="" link class="mr-[5px]">{{ $t('filter.clearTxt') }}</el-button>
                </div>
            </div>
        </el-form>
    </div>
</template>
<script setup lang="ts">
    import { cloneDeep } from '@pureadmin/utils'
    import { useApi } from '@/api'
    import { useRenderIcon } from '@/components/ReIcon/src/hooks'
    import filterIcon from '~icons/fa-solid/filter'
    import infoIcon from '~icons/fa-solid/info-circle'
    import removeIcon from '~icons/ep/close-bold'
    import { useI18n } from 'vue-i18n'
    const { t } = useI18n()
    // 类型定义
    interface FormItem {
        prop: string
        label: string
        type: 'input' | 'select' | 'selectGroup' | 'treeSelect' | 'date' | 'dateMonth' | 'dateDay'
        list?: Array<{
            label: string
            value: string | number
            subFilters?: FormItem[]
            isShowClose?: boolean
            toggle?: boolean
        }>
        rules?: any[]
        tooltip?: string
        itemWidth?: string
        placeholder?: string
        disabled?: boolean
        clearable?: boolean
        multiple?: boolean
        limit?: number
        isShowClose?: boolean
        changeRender?: () => void
        blockRequest?: boolean
        prevProp?: string
        value?: any
        index?: number
        apiUrl?: string
        labelKey?: string
        valueKey?: string
        maxlength?: string | number
        apiParams?: object
        isAsync?: boolean
        defaultValue?: any
        canClear?: boolean
    }

    // 组件参数
    const props = defineProps({
        formInline: { type: Object, default: () => ({}) },
        rules: { type: Object, default: () => ({}) },
        filterList: { type: Array, default: () => [] },
        isDefaultFixed: { type: Boolean, default: false }
    })
    // 组件 emits
    const emit = defineEmits(['update-search', 'update:formInline'])
    // 响应式状态
    const autoWidth = ref('auto')
    const formData = ref<Record<string, any>>({})
    const formList = ref<FormItem[]>([])
    const filterOptsList = ref<FormItem[]>(cloneDeep(props.filterList as FormItem[]))
    const selectKey = ref('')
    const isFilter = ref(true)
    const timer = ref<NodeJS.Timeout | null>(null)
    const lastFormListSubset = ref<FormItem[]>([])
    const formErrors = ref<Record<string, string>>({})
    const isShowClearBtn = computed(() => {
        return formList.value.filter((item: FormItem) => !item.isShowClose).length > 0 && isFilter.value
    })
    // 获取组件实例
    const instance = getCurrentInstance()

    const pickerOptions = (time: any) => {
        const today = new Date()
        today.setHours(0, 0, 0, 0)
        const yesterday = new Date(today.getTime() - 24 * 60 * 60 * 1000)
        return time.getTime() > yesterday.getTime()
    }
    const pickerOptionsForMonth = (time: any) => {
        const now = new Date()
        const firstDayOfCurrentMonth = new Date(now.getFullYear(), now.getMonth(), 1)
        const lastDayOfLastMonth = new Date(firstDayOfCurrentMonth.getTime() - 24 * 60 * 60 * 1000)
        return time.getTime() > lastDayOfLastMonth.getTime()
    }

    const handleSelectChange = async (val: string, toggle = true) => {
        const target = filterOptsList.value.find((item: FormItem) => item.prop === val)
        if (!target) return
        if (target.type === 'select') target.defaultValue = 'all'
        else target.defaultValue = ''

        // 若存在则处理select类型的异步加载
        if (target.apiUrl) {
            setSelectOption(target)
        }

        filterOptsList.value = filterOptsList.value.filter((item: FormItem) => item.prop !== val)
        const merged = [...formList.value, target].filter(Boolean)
        formList.value = [...new Map(merged.map((item: FormItem) => [item.prop, item])).values()]
        isFilter.value = true
        selectKey.value = ''
        if (toggle) {
            await nextTick(() => focusToggle({ ...target, currentRef: `${target.prop}Ref` }))
        }
    }
    const setSelectOption = async (target: FormItem) => {
        await useApi({ name: target.apiUrl, data: target.apiParams }).then((res: any) => {
            if (target.apiUrl === 'getVersionTree') {
                target.list = res.data.children.map((item: any) => ({
                    label: item,
                    value: item
                }))
            } else {
                const list = res.data?.list ?? res.data ?? []
                target.list = list.map((item: any) => ({
                    label: item[target.labelKey],
                    value: item[target.valueKey]
                }))
            }

            target.list.unshift({
                label: t('page.all'),
                value: 'all'
            })
        })

        if (target.defaultValue) {
            formData.value[target.prop] = target.defaultValue ?? ''
            handleDateChange(target)
        }
    }
    const handleCurrentChange = (item: FormItem, toggle = true) => {
        // 1. 执行changeRender（如果存在）
        if (item.changeRender) item.changeRender()

        // 2. 清除定时器
        if (timer.value) clearTimer()

        // 3. 获取当前选中的选项
        const currentValue = formData.value[item.prop]
        const curOption = item.list?.find((option: any) => option.value === currentValue)

        // 4. 处理子过滤器逻辑
        handleSubFilters(item, curOption, curOption?.toggle ?? toggle)

        // 5. 检查是否需要阻止查询请求
        if (item.blockRequest && !formData.value[item.prop]) return

        // 6. 清空时间范围（如果存在）并执行查询
        if (formData.value.timeRange) formData.value.timeRange = []
        handleQuery()
    }

    /**
     * 处理子过滤器逻辑
     */
    const handleSubFilters = (item: FormItem, curOption: Recordable, toggle: boolean) => {
        // 情况1：当前选项有子过滤器
        if (curOption?.subFilters) {
            lastFormListSubset.value = curOption.subFilters

            // 清空第一个子过滤器的值（如果存在）
            const firstSubFilterProp = curOption.subFilters[0]?.prop
            if (firstSubFilterProp && formData.value[firstSubFilterProp]) {
                formData.value[firstSubFilterProp] = ''
            }

            handleTreatment(curOption.subFilters, toggle)
            return
        }

        // 情况2：当前选项存在但没有子过滤器
        if (curOption) {
            handleNoSubFilters(item, curOption)
            updateFormInline(formData.value)
            return
        }

        // 情况3：没有选中任何选项
        lastFormListSubset.value = []
    }

    /**
     * 处理没有子过滤器的情况
     */
    const handleNoSubFilters = (item: FormItem, curOption: Recordable) => {
        const subFilters = handleFindSubFilters(item.list)

        // 移除旧的子过滤器
        if (subFilters?.length) {
            const target = subFilters[0]
            formList.value = formList.value.filter((formItem: FormItem) => formItem.prop !== target.prop)
        }

        // 更新当前表单项的值
        formData.value[item.prop] = curOption.value ?? ''
    }

    const handleTreatment = (subFilters: FormItem[], toggle = true) => {
        const prev = subFilters[0]?.prevProp

        if (!prev) {
            formList.value.push(...subFilters)
            return
        }

        const prevIndex = formList.value.findIndex((item: FormItem) => item.prop === prev)

        if (prevIndex === -1) {
            formList.value.push(...subFilters)
            return
        }

        // 缓存现有 formList 的 prop -> index 映射
        const propIndexMap = new Map<string, number>()
        formList.value.forEach((item, idx) => {
            propIndexMap.set(item.prop, idx)
        })

        const newSubFilters = []

        subFilters.forEach((subItem) => {
            const existingIndex = propIndexMap.get(subItem.prop)

            if (existingIndex !== undefined) {
                // 合并现有项
                formList.value[existingIndex] = {
                    ...formList.value[existingIndex],
                    ...subItem
                }
            } else {
                newSubFilters.push(subItem)
            }
        })

        // 批量插入新项
        if (newSubFilters.length > 0) {
            formList.value.splice(prevIndex + 1, 0, ...newSubFilters)
        }

        toggle &&
            nextTick(() => {
                focusToggle({ ...subFilters[0], currentRef: `${subFilters[0]?.prop}Ref` })
            })
    }

    const handleDateChange = (item: FormItem) => {
        //日期change后，如果是自定义并且清空时间操作，则删除日期弹窗默认选中全部
        if (item.prop === 'timeRange' && item?.prevProp === 'statisticsCycle')
            if (!formData.value.statisticsCycle && !formData.value.timeRange) {
                handleCloseTarget(item)
            }
        emit('update:formInline', { ...props.formInline, ...formData.value })
        handleQuery()
    }

    const onSubmit = (refVal: string) => {
        isFilter.value = false
        nextTick(() => focusToggle(refVal))
    }

    const focusToggle = (refVal: string | { type?: string; currentRef?: string }) => {
        if (typeof refVal === 'string') {
            const refEl = instance?.refs[refVal] as any
            refEl?.focus?.()
            refEl?.toggleMenu?.()
            return
        }
        const type = refVal?.type
        const currentRef = refVal?.currentRef
        if (!currentRef) return
        let target = instance?.refs[currentRef]?.[0]
        if (type) {
            switch (type) {
                case 'date':
                case 'dateDay':
                case 'dateMonth':
                    target?.handleOpen()
                    break
                case 'select':
                case 'selectGroup':
                case 'treeSelect':
                    target?.toggleMenu?.()
                    handleCurrentChange(refVal as FormItem)
                    break
                default:
                    target?.focus?.()
                    break
            }
        }
    }

    const handleCloseTarget = (target: FormItem, flag = false) => {
        delete formErrors.value[target.prop]
        if (lastFormListSubset.value.includes(target)) {
            lastFormListSubset.value = lastFormListSubset.value.filter((item: any) => item !== target)
        }

        const { subFilters } = target.list?.find((item: any) => item.subFilters?.length) ?? {}
        if (subFilters?.length) {
            subFilters.forEach((item: any) => {
                if (lastFormListSubset.value.includes(item)) {
                    formData.value[item.prop] = item.defaultValue ?? ''
                    lastFormListSubset.value = lastFormListSubset.value.filter((i: any) => i !== item)
                }
                formList.value = formList.value.filter((i: FormItem) => i.prop !== item.prop)
            })
        }

        formList.value = formList.value.filter((item: FormItem) => item.prop !== target.prop)
        setFormItemDefaultValue(target)
        updateFormInline(formData.value)
        const existKey = formList.value.map((i: FormItem) => i.prop)
        const filterList = cloneDeep(props.filterList)
        filterOptsList.value = filterList.filter((i: FormItem) => !existKey.includes(i.prop)) as FormItem[]
        //删除后清空相应defaultValue默认值
        const targetItem = filterOptsList.value.find((item) => item.prop === target.prop)
        if (targetItem?.defaultValue) {
            if (targetItem.type === 'select') targetItem.defaultValue = 'all'
            else targetItem.defaultValue = ''
        }
        if (target.prop === 'timeRange') {
            formData.value[target.prevProp] = 'all'
        }
        if (!flag) handleQuery()
    }
    const setFormItemDefaultValue = (item: FormItem) => {
        formData.value[item.prop] = item.defaultValue && !item.canClear ? item.defaultValue : ''
    }
    const clearForm = () => {
        if (!props.isDefaultFixed) formErrors.value = {}
        formList.value = formList.value.filter((item: FormItem) => item.isShowClose)
        formList.value.forEach((item) => {
            if (item.list) {
                const replaceIndex = formList.value.findIndex((key) => key.prevProp === item.prop)
                formList.value[replaceIndex] = {
                    ...formList.value[replaceIndex],
                    ...item.list[0]?.subFilters?.[0]
                }
            }
        })
        filterOptsList.value = cloneDeep(props.filterList).filter((item: FormItem) => !item.isShowClose) as FormItem[]
        isFilter.value = true

        clearFormData(filterOptsList.value)
        clearFormData(formList.value)

        updateFormInline(formData.value)
    }
    const clearFormData = (formList: Recordable[]) => {
        formList?.forEach((item: FormItem) => {
            setFormItemDefaultValue(item)
            if (item?.list?.length) {
                item.list.forEach((i: Recordable) => {
                    i?.subFilters?.forEach((j: Recordable) => {
                        formData.value[j.prop] = j.defaultValue ?? ''
                    })
                })
            }
        })
    }
    const handleClear = () => {
        clearForm()
        handleQuery(true)
    }
    const handleClearExport = () => {
        clearForm()
        emit('update-search', false)
    }
    const updateFormInline = (nw: Recordable) => {
        emit('update:formInline', { ...props.formInline, ...nw })
    }
    const handleQuery = (flag = false) => {
        if (timer.value) clearTimer()
        if (!Object.keys(formErrors.value).length) {
            openTimer(flag)
        }
    }

    const openTimer = (flag = false) => {
        if (timer.value) clearTimer()
        timer.value = setTimeout(() => {
            emit('update-search', flag)
        }, 1500)
    }

    const clearTimer = () => {
        if (timer.value) {
            clearTimeout(timer.value)
            timer.value = null
        }
    }

    const handleVisible = (val: boolean) => (isFilter.value = !val)

    const handleSetFormList = () => {
        const filterData = cloneDeep(props.filterList)
        const params: Record<string, any> = {}

        Object.keys(formData.value).forEach((key: string) => {
            if ((formData.value[key] && formData.value[key] !== 'all') || formData.value[key] === 0) {
                params[key] = formData.value[key]
            }
        })
        updateFormInline(formData.value)
        if (!params.timeRange?.length) delete params.timeRange
        let keys = Object.keys(params)
        filterOptsList.value = filterData.filter((item: FormItem) => !keys.includes(item.prop)) as FormItem[]

        formList.value = filterData.filter((item: FormItem) => {
            const baseCondition = keys.includes(item.prop) || item.isShowClose
            const excludeAll = item.defaultValue !== 'all'
            return baseCondition && excludeAll
        }) as FormItem[]

        formList.value.forEach(async (item: FormItem) => {
            if (item.apiUrl) {
                setSelectOption(item)
            }
            if (item.isShowClose && item.list) {
                const hasIsShowCloseTrue = item.list.some((key) => key.isShowClose)
                if (hasIsShowCloseTrue) handleCurrentChange(item, false)
            }
        })

        isFilter.value = !!filterOptsList.value.length
    }

    const handleValidate = (prop: string, isValid: boolean, message: string) => {
        if (isValid) {
            delete formErrors.value[prop]
            handleQuery()
        } else {
            if (timer.value) clearTimer()
            formErrors.value[prop] = message
        }
    }

    const handleShowBlockItem = () => {
        filterOptsList.value.forEach((item: FormItem) => {
            if (item.isShowClose && !item.isAsync) {
                handleSelectChange(item.prop, false)
                if (item.list?.length) handleCurrentChange(item, false)
            }
        })
    }
    function handleFindSubFilters(list: Recordable[]) {
        const subFilters = list.map((item: Recordable) => item.subFilters).filter(Boolean)
        return subFilters.flat()
    }

    // 生命周期
    watch(
        () => props.formInline,
        (nw: Record<string, any>) => {
            if (nw) formData.value = nw
        },
        { deep: true, immediate: true }
    )
    watch(
        () => props.filterList,
        (nw: FormItem[]) => {
            if (nw) {
                filterOptsList.value = nw
                handleSetFormList()
            }
        },
        { deep: true, immediate: true }
    )

    onMounted(() => nextTick(() => handleShowBlockItem()))

    onBeforeUnmount(() => clearTimer())
    defineExpose({ handleClearExport, handleSelectChange })
</script>

<style scoped lang="scss">
    .search-box {
        box-sizing: border-box;
        margin-top: 13px;
        .el-form {
            .flex-warp {
                flex-wrap: wrap;
                position: relative;
                min-height: 36px;
                .el-form-item {
                    margin-bottom: 5px;
                    margin-right: 2px;
                    &.close-box {
                        .close-btn {
                            font-size: 16px !important;
                            visibility: hidden;
                        }
                        &:hover {
                            .close-btn {
                                visibility: visible;
                            }
                        }
                    }
                    .visually-hidden {
                        visibility: hidden;
                    }
                    .el-input,
                    .el-select {
                        width: 230px;
                    }
                    .el-date-editor {
                        width: 320px;
                        margin-top: 1px;
                        .el-range-input {
                            width: 40%;
                        }
                        .el-range__icon {
                            margin-left: -8px;
                        }
                        .el-range__close-icon {
                            width: 20px;
                        }
                    }
                    .el-form-item__error {
                        display: none;
                    }
                    .icon-error {
                        position: absolute;
                        right: 15px;
                        top: 8px;
                    }
                    &.is-error {
                        .close-btn {
                            margin-left: 18px;
                        }
                    }
                }
                .filter-select {
                    width: 200px;
                }
                .clear-btn {
                    position: absolute;
                    right: 10px;
                    bottom: 10px;
                    z-index: 32;
                    visibility: hidden;
                    cursor: pointer;
                }
                &:hover {
                    .clear-btn {
                        visibility: visible;
                    }
                }
            }
        }
    }
</style>
