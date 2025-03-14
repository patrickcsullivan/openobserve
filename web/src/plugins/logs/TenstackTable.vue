<template>
  <div ref="parentRef" class="container tw-overflow-x-auto tw-relative">
    <table
      data-test="logs-search-result-logs-table"
      class="tw-w-full tw-table-auto"
      :style="{
        minWidth: '100%',
        minHeight: totalSize + 'px',
        ...columnSizeVars,
        width: !columnOrder.includes('source')
          ? table.getCenterTotalSize() + 'px'
          : wrap
            ? width
              ? width - 12 + 'px'
              : '100%'
            : '100%',
      }"
    >
      <thead class="tw-sticky tw-top-0 tw-z-50">
        <vue-draggable
          v-model="columnOrder"
          v-for="headerGroup in table.getHeaderGroups()"
          :key="headerGroup.id"
          :element="'table'"
          :animation="200"
          :sort="!isResizingHeader || !columnOrder.includes('source')"
          handle=".table-head"
          :class="{
            'tw-cursor-move': table.getState().columnOrder.length > 1,
          }"
          :style="{
            width:
              columnOrder.includes('source') && wrap
                ? width - 12 + 'px'
                : tableRowSize + 'px',
            minWidth: '100%',
            background: store.state.theme === 'dark' ? '#565656' : '#F5F5F5',
          }"
          tag="tr"
          @start="(event) => handleDragStart(event)"
          @end="(event) => handleDragEnd(event)"
          class="tw-flex items-center"
        >
          <th
            v-for="header in headerGroup.headers"
            :key="header.id"
            :id="header.id"
            class="tw-px-2 tw-relative table-head tw-text-ellipsis"
            :style="{ width: header.getSize() + 'px' }"
            :data-test="`log-search-result-table-th-${header.id}`"
          >
            <div
              v-if="header.column.getCanResize()"
              @dblclick="header.column.resetSize()"
              @mousedown.self.prevent.stop="header.getResizeHandler()?.($event)"
              @touchstart.self.prevent.stop="
                header.getResizeHandler()?.($event)
              "
              :class="[
                'resizer',
                header.column.getIsResizing() ? 'isResizing' : '',
              ]"
              :style="{}"
              class="tw-right-0"
            />

            <div
              v-if="!header.isPlaceholder"
              :class="[
                'text-left',
                header.column.getCanSort() ? 'cursor-pointer select-none' : '',
              ]"
              @click="
                getSortingHandler(
                  $event,
                  header.column.getToggleSortingHandler(),
                )
              "
              class="tw-overflow-hidden tw-text-ellipsis"
            >
              <FlexRender
                :render="header.column.columnDef.header"
                :props="header.getContext()"
              />

              <div
                :data-test="`log-add-data-from-column-${header.column.columnDef.header}`"
                class="tw-invisible tw-items-center tw-absolute tw-right-2 tw-top-0 tw-px-2 column-actions"
                :class="
                  store.state.theme === 'dark' ? 'field_overlay_dark' : ''
                "
                v-if="
                  (header.column.columnDef.meta as any).closable ||
                  (header.column.columnDef.meta as any).showWrap
                "
              >
                <!-- <span
                        v-if="(header.column.columnDef.meta as any).showWrap"
                        style="font-weight: normal"
                        :class="
                          store.state.theme === 'dark'
                            ? 'text-white'
                            : 'text-grey-9'
                        "
                        >{{ t("common.wrap") }}</span
                      >
                      <q-toggle
                        v-if="(header.column.columnDef.meta as any).showWrap"
                        class="text-normal q-ml-xs q-mr-sm"
                        :data-test="`logs-search-result-table-th-remove-${header.column.columnDef.header}-btn`"
                        v-model="(header.column.columnDef.meta as any).wrapContent"
                        color="primary"
                        :class="
                          store.state.theme === 'dark'
                            ? 'text-white'
                            : 'text-grey-7'
                        "
                        size="xs"
                        dense
                      /> -->

                <q-icon
                  v-if="(header.column.columnDef.meta as any).closable"
                  :data-test="`logs-search-result-table-th-remove-${header.column.columnDef.header}-btn`"
                  name="cancel"
                  class="q-ma-none close-icon cursor-pointer"
                  :class="
                    store.state.theme === 'dark' ? 'text-white' : 'text-grey-7'
                  "
                  :title="t('common.close')"
                  size="18px"
                  @click="closeColumn(header.column.columnDef)"
                >
                </q-icon>
              </div>
            </div>
          </th>
        </vue-draggable>

        <tr v-if="loading" class="tw-w-full">
          <td
            :colspan="columnOrder.length"
            class="text-bold"
            style="opacity: 0.7"
          >
            <div
              class="text-subtitle2 text-weight-bold tw-flex tw-items-center"
            >
              <q-spinner-hourglass size="20px" />
              {{ t("confirmDialog.loading") }}
            </div>
          </td>
        </tr>
        <tr v-if="!loading && errMsg != ''" class="tw-w-full">
          <td
            :colspan="columnOrder.length"
            class="text-bold"
            style="opacity: 0.7"
          >
            <div class="text-subtitle2 text-weight-bold bg-warning">
              <q-icon size="xs" name="warning" class="q-mr-xs" />
              {{ errMsg }}
            </div>
          </td>
        </tr>
      </thead>
      <tr
        data-test="log-search-result-function-error"
        v-if="functionErrorMsg != ''"
      >
        <td
          :colspan="columnOrder.length"
          class="text-bold"
          style="opacity: 0.6"
        >
          <div class="text-subtitle2 text-weight-bold bg-warning">
            <q-btn
              :icon="
                expandedRowIndices.includes(-1)
                  ? 'expand_more'
                  : 'chevron_right'
              "
              dense
              size="xs"
              flat
              class="q-mr-xs"
              data-test="table-row-expand-menu"
              @click.self.stop="expandRow(-1)"
            ></q-btn
            ><b>
              <q-icon name="warning" size="15px"></q-icon>
              {{ t("search.functionErrorLabel") }}</b
            >
          </div>
        </td>
      </tr>
      <tr v-if="expandedRowIndices.includes(-1)">
        <td
          :colspan="columnOrder.length"
          class="bg-warning"
          style="opacity: 0.7"
        >
          <pre>{{ functionErrorMsg }}</pre>
        </td>
      </tr>
      <tbody
        data-test="logs-search-result-table-body"
        ref="tableBodyRef"
        class="tw-relative"
      >
        <template v-for="virtualRow in virtualRows" :key="virtualRow.id">
          <tr
            :data-test="`logs-search-result-detail-${
              (tableRows[virtualRow.index] as any)[
                store.state.zoConfig.timestamp_column || '_timestamp'
              ]
            }`"
            :style="{
              transform: `translateY(${virtualRow.start}px)`,
              minWidth: '100%',
            }"
            :data-index="virtualRow.index"
            :ref="(node: any) => rowVirtualizer.measureElement(node)"
            class="tw-absolute tw-flex tw-w-max tw-items-center tw-justify-start tw-border-b tw-cursor-pointer"
            :class="[
              store.state.theme === 'dark'
                ? 'w-border-gray-800  hover:tw-bg-zinc-800'
                : 'w-border-gray-100 hover:tw-bg-zinc-100',
              columnOrder.includes('source') && !wrap
                ? 'tw-table-row'
                : 'tw-flex',
            ]"
            @click="
              !(formattedRows[virtualRow.index]?.original as any)
                ?.isExpandedRow &&
                handleDataRowClick(
                  tableRows[virtualRow.index],
                  virtualRow.index,
                )
            "
          >
            <td
              v-if="
                (formattedRows[virtualRow.index]?.original as any)
                  ?.isExpandedRow
              "
              :colspan="columnOrder.length"
              :data-test="`log-search-result-expanded-row-${virtualRow.index}`"
            >
              <json-preview
                :value="tableRows[virtualRow.index - 1] as any"
                show-copy-button
                class="tw-py-1"
                @copy="copyLogToClipboard"
                @add-field-to-table="addFieldToTable"
                @add-search-term="addSearchTerm"
              />
            </td>
            <template v-else>
              <td
                v-for="(cell, cellIndex) in formattedRows[
                  virtualRow.index
                ].getVisibleCells()"
                :key="cell.id"
                :data-test="
                  'log-table-column-' +
                  virtualRow.index +
                  '-' +
                  cell.column.columnDef.id
                "
                class="tw-py-1 tw-px-2 tw-items-center tw-justify-start tw-relative table-cell"
                :style="{
                  width:
                    cell.column.columnDef.id !== 'source'
                      ? cell.column.getSize() + 'px'
                      : wrap
                        ? width - 225 - 12 + 'px'
                        : 'auto',
                  height: wrap ? '100%' : '26px',
                }"
                :class="[
                  columnOrder.includes('source') && !wrap
                    ? 'tw-table-cell'
                    : 'tw-block',
                  !wrap &&
                    'tw-overflow-hidden tw-text-ellipsis tw-whitespace-nowrap',
                  wrap && ' tw-break-words',
                ]"
              >
                <q-btn
                  v-if="cellIndex == 0"
                  :icon="
                    expandedRowIndices.includes(virtualRow.index)
                      ? 'expand_more'
                      : 'chevron_right'
                  "
                  dense
                  size="xs"
                  flat
                  class="q-mr-xs"
                  data-test="table-row-expand-menu"
                  @click.capture.stop="handleExpandRow(virtualRow.index)"
                ></q-btn>

                <cell-actions
                  v-if="
                    (cell.column.columnDef.meta as any)?.closable &&
                    (cell.row.original as any)[cell.column.id]
                  "
                  :column="cell.column"
                  :row="cell.row.original as any"
                  @copy="copyLogToClipboard"
                  @add-search-term="addSearchTerm"
                  @add-field-to-table="addFieldToTable"
                />
                <FlexRender
                  :render="cell.column.columnDef.cell"
                  :props="cell.getContext()"
                />
              </td>
            </template>
          </tr>
        </template>
      </tbody>
    </table>
  </div>
</template>

<script setup lang="ts">
import { ref, computed, defineEmits, watch, nextTick, onMounted } from "vue";
import { useVirtualizer } from "@tanstack/vue-virtual";
import {
  FlexRender,
  type ColumnDef,
  type SortingState,
  useVueTable,
  getCoreRowModel,
  getSortedRowModel,
} from "@tanstack/vue-table";
import JsonPreview from "./JsonPreview.vue";
import { useStore } from "vuex";
import { useI18n } from "vue-i18n";
import { VueDraggableNext as VueDraggable } from "vue-draggable-next";
import CellActions from "@/plugins/logs/data-table/CellActions.vue";

const props = defineProps({
  rows: {
    type: Array,
    required: true,
  },
  columns: {
    type: Array,
    required: true,
  },
  wrap: {
    type: Boolean,
    default: false,
  },
  width: {
    type: Number,
    default: 56,
  },
  loading: {
    type: Boolean,
    default: false,
  },
  errMsg: {
    type: String,
    default: "",
  },
  functionErrorMsg: {
    type: String,
    default: "",
  },
  expandedRows: {
    type: Array,
    default: () => [],
  },
});

const { t } = useI18n();

const emits = defineEmits([
  "copy",
  "addSearchTerm",
  "addFieldToTable",
  "closeColumn",
  "click:dataRow",
  "update:columnSizes",
  "update:columnOrder",
  "expandRow",
]);

const sorting = ref<SortingState>([]);

const store = useStore();

const getSortingHandler = (e: Event, fn: any) => {
  return fn(e);
};

const setSorting = (sortingUpdater: any) => {
  const newSortVal = sortingUpdater(sorting.value);

  sorting.value = newSortVal;
};

const tableBodyRef = ref<any>(null);

const columnResizeMode = "onChange";

const tableRowSize = ref(0);

const columnOrder = ref<any>([]);

const tableRows = ref(props.rows);

watch(
  () => props.columns,
  (newVal) => {
    columnOrder.value = newVal.map((column: any) => column.id);
  },
  {
    deep: true,
  },
);

watch(
  () => props.rows,
  async (newVal) => {
    if (newVal) tableRows.value = [...newVal];

    await nextTick();
    await nextTick();

    expandedRowIndices.value = [];
    props.expandedRows.forEach((index) => {
      expandRow(index as number);
    });
  },
  {
    deep: true,
  },
);

const table = useVueTable({
  get data() {
    return tableRows.value || [];
  },
  get columns() {
    return props.columns as ColumnDef<unknown, any>[];
  },
  state: {
    get sorting() {
      return sorting.value;
    },
    get columnOrder() {
      emits("update:columnOrder", columnOrder.value);
      return columnOrder.value;
    },
  },
  onSortingChange: setSorting,
  getCoreRowModel: getCoreRowModel(),
  getSortedRowModel: getSortedRowModel(),
  defaultColumn: {
    minSize: 60,
    maxSize: 800,
  },
  debugTable: true,
  debugHeaders: true,
  debugColumns: true,
  columnResizeMode,
  enableColumnResizing: true,
  onStateChange: async (state) => {
    await nextTick();
    tableRowSize.value = tableBodyRef.value.children[0]?.scrollWidth;
  },
});

const columnSizeVars = computed(() => {
  const headers = table.getFlatHeaders();
  const colSizes: { [key: string]: number } = {};
  for (let i = 0; i < headers.length; i++) {
    const header = headers[i]!;
    colSizes[`--header-${header.id}-size`] = header.getSize();
    colSizes[`--col-${header.column.id}-size`] = header.column.getSize();
  }
  return colSizes;
});

watch(columnSizeVars, (newColSizes) => {
  emits("update:columnSizes", newColSizes);
});

onMounted(() => {
  props.expandedRows.forEach((index) => {
    expandRow(index as number);
  });
});

const formattedRows = computed(() => {
  return table.getRowModel().rows;
});

const isResizingHeader = ref(false);

const headerGroups = computed(() => table.getHeaderGroups()[0]);

const headers = computed(() => headerGroups.value.headers);

watch(
  () => headers.value,
  (newVal) => {
    isResizingHeader.value = newVal.some((header) =>
      header.column.getIsResizing(),
    );
  },
  {
    deep: true,
  },
);

const parentRef = ref<HTMLElement | null>(null);

const rowVirtualizerOptions = computed(() => {
  return {
    count: formattedRows.value.length,
    getScrollElement: () => parentRef.value,
    estimateSize: () => 26,
    overscan: 5,
    measureElement:
      typeof window !== "undefined" &&
      navigator.userAgent.indexOf("Firefox") === -1
        ? (element: any) => element?.getBoundingClientRect().height
        : undefined,
  };
});

const rowVirtualizer = useVirtualizer(rowVirtualizerOptions);

const virtualRows = computed(() => rowVirtualizer.value.getVirtualItems());

const totalSize = computed(() => rowVirtualizer.value.getTotalSize());

const copyLogToClipboard = (value: any) => {
  emits("copy", value);
};
const addSearchTerm = (value: string) => {
  emits("addSearchTerm", value);
};
const addFieldToTable = (value: string) => {
  emits("addFieldToTable", value);
};

const closeColumn = (data: any) => {
  emits("closeColumn", data);
};

const handleDragStart = (event: any) => {
  if (columnOrder.value[event.oldIndex] === "@timestamp") {
    isResizingHeader.value = true;
  } else {
    isResizingHeader.value = false;
  }
};

const handleDragEnd = async (event: any) => {
  if (
    columnOrder.value.includes("@timestamp") &&
    columnOrder.value[0] !== "@timestamp"
  ) {
    await nextTick();
    const newItem = columnOrder.value[event.newIndex];
    columnOrder.value[event.newIndex] = columnOrder.value[event.oldIndex];
    columnOrder.value[event.oldIndex] = newItem;

    columnOrder.value = [...columnOrder.value];
  }
};

const expandedRowIndices = ref<number[]>([]);

const handleExpandRow = (index: number) => {
  emits("expandRow", calculateActualIndex(index));

  expandRow(index);
};

const expandRow = async (index: number) => {
  let isCollapseOperation = false;

  if (expandedRowIndices.value.includes(index)) {
    expandedRowIndices.value = expandedRowIndices.value.filter(
      (i) => i !== index,
    );

    expandedRowIndices.value = expandedRowIndices.value.map((i) =>
      i > index ? i - 1 : i,
    );

    tableRows.value.splice(index + 1, 1);
    isCollapseOperation = true;
  } else {
    expandedRowIndices.value.push(index);
    tableRows.value.splice(index + 1, 0, {
      isExpandedRow: true,
      ...(props.rows[index] as {}),
    });

    expandedRowIndices.value = expandedRowIndices.value.map((i) =>
      i > index ? i + 1 : i,
    );
  }

  expandedRowIndices.value = expandedRowIndices.value.sort();

  tableRows.value = [...tableRows.value];

  await nextTick();

  if (isCollapseOperation)
    expandedRowIndices.value.forEach((expandedIndex) => {
      if (expandedIndex !== -1) {
        formattedRows.value[expandedIndex].toggleExpanded();
      }
    });
};

const calculateActualIndex = (index: number): number => {
  let actualIndex = index;
  expandedRowIndices.value.forEach((expandedIndex) => {
    if (expandedIndex !== -1 && expandedIndex < index) {
      actualIndex -= 1;
    }
  });
  return actualIndex;
};

const handleDataRowClick = (row: any, index: number) => {
  const actualIndex = calculateActualIndex(index);

  if (actualIndex !== -1) {
    emits("click:dataRow", row, actualIndex);
  }
};

defineExpose({
  parentRef,
});
</script>
<style scoped lang="scss">
.resizer {
  position: absolute;
  top: 0;
  height: 100%;
  width: 5px;
  background: rgba(0, 0, 0, 0.5);
  cursor: col-resize;
  user-select: none;
  touch-action: none;
}

.resizer.ltr {
  right: 0;
}

.resizer.rtl {
  left: 0;
}

.resizer.isResizing {
  background: blue;
  opacity: 1;
}

@media (hover: hover) {
  .resizer {
    opacity: 0;
  }

  *:hover > .resizer {
    opacity: 1;
  }
}

.container {
  height: calc(100vh - 294px);
  overflow: auto;
}

.cursor-pointer {
  cursor: pointer;
}

.select-none {
  user-select: none;
}

.text-left {
  text-align: left;
}

table {
  border-collapse: collapse;
  border-spacing: 0;
  font-size: 12px !important;
}

thead {
  background: lightgray;
  font-family: "Nunito Sans", sans-serif;
  font-size: 14px !important;
}

.table-row {
  border-bottom: 1px solid gray;
}

th {
  text-align: left;

  &:hover {
    .column-actions {
      visibility: visible !important;
    }
  }
}

td {
  font-family: monospace;
}

.thead-sticky tr > *,
.tfoot-sticky tr > * {
  position: sticky;
  opacity: 1;
  z-index: 1;
  background: #f5f5f5;
}

.q-table--dark .thead-sticky tr > *,
.q-table--dark .tfoot-sticky tr > * {
  background: #565656;
}

.table-cell {
  .table-cell-actions {
    visibility: hidden !important;
  }

  &:hover {
    .table-cell-actions {
      visibility: visible !important;
    }
  }
}
</style>
