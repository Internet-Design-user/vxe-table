<template>
    <el-container class="h-full computed-height bg-bg_color">
        <el-header height="auto">
            <ReFilter
                ref="filterRef"
                v-if="filterList.length"
                :filterList="filterList"
                :formInline="formInline"
                @update:formInline="updateFormInline"
                @update-search="handleUpdateSearch"
            />
            <div class="flex justify-start w-full bg-bg_color operate-btn">
                <vxe-toolbar>
                    <template #toolSuffix>
                        <el-button type="text" :icon="fa-refresh" @click="handleSearch">{{ $t('page.refresh') }}</el-button>
                        <el-button type="text" icon="fa-cog" @click="openCustomEvent">{{ $t('page.columnSet') }}</el-button>
                    </template>
                </vxe-toolbar>
                <vxe-toolbar class="ml-4">
                    <template #toolSuffix>
                        <template v-for="item in toolbarTools" :key="item.id">
                            <el-button v-if="!item.tooltip" type="text" :icon="item.icon" @click="item.callback">
                                {{ item.label }}
                            </el-button>
                            <!-- 有tooltip字段时显示提示框 -->
                            <el-tooltip v-else :content="item.tooltip" placement="top" effect="dark">
                                <el-button type="text" :icon="item.icon" @click="item.callback">
                                    {{ item.label }}
                                </el-button>
                            </el-tooltip>
                        </template>
                    </template>
                </vxe-toolbar>
            </div>
        </el-header>
        <el-main class="p-4! pt-0! pb-1!">
            <vxe-table
                ref="tableRef"
                border
                auto-resize
                height="100%"
                :loading="loading"
                :column-config="columnConfig"
                :column-drag-config="columnDragConfig"
                :row-config="rowConfig"
                :sort-config="sortConfig"
                :checkbox-config="{ highlight: true }"
                :data="tableData"
                :custom-config="customConfig"
                :cell-config="{ height: 36 }"
                :resizable-config="resizableConfig"
                @checkbox-change="handleCheckboxChange"
                @checkbox-all="handleCheckboxAllChange"
            >
                >
                <vxe-column v-if="showSelection" type="checkbox" width="60" field="checkbox" fixed="left" />
                <vxe-column v-if="showIndex" type="seq" width="70" flxed="left" />
                <!--field 类型为 string|(index:number)=>string   特殊处理-->
                <vxe-column
                    v-for="item in tableColumns"
                    show-header-overflow
                    :key="item.prop"
                    :fixed="item.fixed === true ? 'left' : (item.fixed as 'left' | 'right')"
                    :width="item.width"
                    :align="item.align || 'left'"
                    :visible="item.visible ?? true"
                    :min-width="item.minWidth"
                    :field="transformField(item.prop)"
                    :title="item.label"
                    :sortable="item.sortable === true"
                    :sort-by="item.sortCustomMethod"
                    :show-overflow="item.showOverflow ?? true"
                    :drag-sort="item.dragSort"
                    :title-prefix="item.titlePrefix"
                    :title-suffix="item.titleSuffix"
                >
                    <template #default="{ row }">
                        <render-cell :row="row" :prop="item.prop" :render="item.render" />
                    </template>
                </vxe-column>
            </vxe-table>
        </el-main>
        <el-footer height="45px" class="flex justify-end mb-2">
            <el-pagination
                v-model:current-page="pageable.pageNum"
                v-model:page-size="pageable.pageSize"
                background
                :page-sizes="sizeArray"
                :layout="layout"
                :total="pageable.total"
                @size-change="handleSizeChange"
                @current-change="handleCurrentChange"
            />
        </el-footer>
    </el-container>
</template>

<script setup lang="ts">
    import useTable from './useTable'
    import RenderCell from './render.vue'
    const props = defineProps({
        filterList: { type: Array, default: () => [] },
        operateGroup: { type: Array, default: () => [] },
        showSelection: { type: Boolean, default: true },
        showIndex: { type: Boolean, default: false },
        columns: { type: Array, default: () => [] },
        apiUrl: { type: String, default: '' },
        paramStruct: { type: Function, default: null },
        layout: {
            type: String,
            default: 'total, sizes, prev, pager, next, jumper'
        },
        sizeArray: {
            type: Array as PropType<number[]>,
            default: () => [20, 50, 100]
        },
        isManualQuery: { type: Boolean, default: true } //是否默认调用查询方法
    })
    const emits = defineEmits<(e: 'checkbox-change', list: any[]) => void>()
    const filterRef = ref(null)
    const {
        tableRef,
        loading,
        formInline,
        tableData,
        pageable,
        tableColumns,
        rowConfig,
        sortConfig,
        columnConfig,
        customConfig,
        toolbarTools,
        resizableConfig,
        columnDragConfig,
        handleUpdateSearch,
        handleSearch,
        handleSizeChange,
        updateFormInline,
        handleCurrentChange,
        handleCheckboxChange,
        handleCheckboxAllChange,
        openCustomEvent,
        transformField
    } = useTable(props, emits)

    watch(
        () => props.columns,
        (nw: TableColumnItem[]) => {
            if (nw) tableColumns.value = nw
        },
        {
            deep: true,
            immediate: true
        }
    )
    const handleFilterClear = () => {
        filterRef.value.handleClearExport()
    }
    const handleFilterSelectChange = (val: string, toggle = true) => {
        filterRef.value.handleSelectChange(val, toggle)
    }
    defineExpose({ handleSearch, handleFilterClear, handleFilterSelectChange })
</script>

<style lang="scss">
    .operate-btn {
        .el-button--text {
            margin-right: 10px;
            [class*='el-icon'] + span {
                margin-left: 3px;
            }
        }
        .vxe-toolbar {
            padding: 8px 0 10px;
        }
    }
</style>
