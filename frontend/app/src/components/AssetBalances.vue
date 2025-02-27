<script setup lang="ts">
import { some } from 'lodash-es';
import { isEvmNativeToken } from '@/types/asset';
import type { AssetBalance, AssetBalanceWithPrice } from '@rotki/common';
import type { DataTableColumn, DataTableSortData } from '@rotki/ui-library';

defineOptions({
  name: 'AssetBalances',
});

const search = defineModel<string>('search', { required: false, default: '' });

const props = withDefaults(
  defineProps<{
    balances: AssetBalanceWithPrice[];
    details?: {
      groupId: string;
      chains: string[];
    };
    loading?: boolean;
    hideTotal?: boolean;
    hideBreakdown?: boolean;
    stickyHeader?: boolean;
    isLiability?: boolean;
    allBreakdown?: boolean;
  }>(),
  {
    loading: false,
    hideTotal: false,
    hideBreakdown: false,
    stickyHeader: false,
    details: undefined,
    isLiability: false,
    allBreakdown: false,
  },
);

const { t } = useI18n();

const { balances } = toRefs(props);
const expanded = ref<AssetBalanceWithPrice[]>([]);

const { assetSymbol, assetName, assetInfo } = useAssetInfoRetrieval();

function assetFilter(item: AssetBalance) {
  const keyword = get(search).toLowerCase()?.trim() ?? '';
  if (!keyword)
    return true;

  const name = get(assetName(item.asset))?.toLowerCase()?.trim();
  const symbol = get(assetSymbol(item.asset))?.toLowerCase()?.trim();

  return symbol.includes(keyword) || name.includes(keyword);
}

const filteredBalances = computed(() => get(balances).filter(assetFilter));

const total = computed(() => bigNumberSum(filteredBalances.value.map(({ usdValue }) => usdValue)));

const { currencySymbol } = storeToRefs(useGeneralSettingsStore());

const sort = ref<DataTableSortData<AssetBalanceWithPrice>>({
  column: 'usdValue',
  direction: 'desc' as const,
});

const tableHeaders = computed<DataTableColumn<AssetBalanceWithPrice>[]>(() => [
  {
    label: t('common.asset'),
    key: 'asset',
    class: 'text-no-wrap w-full',
    cellClass: 'py-0',
    sortable: true,
  },
  {
    label: t('common.price_in_symbol', {
      symbol: get(currencySymbol),
    }),
    key: 'usdPrice',
    align: 'end',
    cellClass: 'py-0',
    sortable: true,
  },
  {
    label: t('common.amount'),
    key: 'amount',
    align: 'end',
    cellClass: 'py-0',
    sortable: true,
  },
  {
    label: t('common.value_in_symbol', {
      symbol: get(currencySymbol),
    }),
    key: 'usdValue',
    align: 'end',
    class: 'text-no-wrap',
    cellClass: 'py-0',
    sortable: true,
  },
]);

const sortItems = getSortItems<AssetBalanceWithPrice>(asset => get(assetInfo(asset)));

const sorted = computed<AssetBalanceWithPrice[]>(() => {
  const sortBy = get(sort);
  const data = [...get(filteredBalances)];
  if (!Array.isArray(sortBy) && sortBy?.column)
    return sortItems(data, [sortBy.column as keyof AssetBalance], [sortBy.direction === 'desc']);

  return data;
});

const isExpanded = (asset: string) => some(get(expanded), { asset });

function expand(item: AssetBalanceWithPrice) {
  set(expanded, isExpanded(item.asset) ? [] : [item]);
}

function getAssets(item: AssetBalanceWithPrice): string[] {
  return item.breakdown?.map(entry => entry.asset) ?? [];
}
</script>

<template>
  <RuiDataTable
    v-model:sort.external="sort"
    :cols="tableHeaders"
    :rows="sorted"
    :loading="loading"
    :expanded="expanded"
    :loading-text="t('asset_balances.loading')"
    :empty="{ description: t('data_table.no_data') }"
    :sticky-header="stickyHeader"
    row-attr="asset"
    single-expand
    outlined
    dense
  >
    <template #item.asset="{ row }">
      <AssetDetails
        opens-details
        :asset="row.asset"
        :is-collection-parent="!!row.breakdown"
      />
    </template>
    <template #item.usdPrice="{ row }">
      <AmountDisplay
        :loading="!row.usdPrice || row.usdPrice.lt(0)"
        no-scramble
        show-currency="symbol"
        :price-asset="row.asset"
        :price-of-asset="row.usdPrice"
        fiat-currency="USD"
        :value="row.usdPrice"
      />
    </template>
    <template #item.amount="{ row }">
      <AmountDisplay :value="row.amount" />
    </template>
    <template #item.usdValue="{ row }">
      <AmountDisplay
        show-currency="symbol"
        :amount="row.amount"
        :price-asset="row.asset"
        :price-of-asset="row.usdPrice"
        fiat-currency="USD"
        :value="row.usdValue"
      />
    </template>
    <template
      v-if="balances.length > 0 && !hideTotal"
      #body.append
    >
      <RowAppend
        label-colspan="3"
        :label="t('common.total')"
        :is-mobile="false"
        :right-patch-colspan="2"
        class-name="[&>td]:p-4 text-sm"
      >
        <AmountDisplay
          fiat-currency="USD"
          show-currency="symbol"
          :value="total"
        />
      </RowAppend>
    </template>
    <template #expanded-item="{ row }">
      <EvmNativeTokenBreakdown
        v-if="!hideBreakdown && isEvmNativeToken(row.asset)"
        :blockchain-only="!allBreakdown"
        :assets="getAssets(row)"
        :details="details"
        :identifier="row.asset"
        :is-liability="isLiability"
        class="bg-white dark:bg-[#1E1E1E] my-2"
      />
      <AssetBalances
        v-else
        v-bind="props"
        hide-total
        :balances="row.breakdown ?? []"
        :sticky-header="false"
        :is-liability="isLiability"
        :all-breakdown="allBreakdown"
        class="bg-white dark:bg-[#1E1E1E] my-2"
      />
    </template>
    <template #item.expand="{ row }">
      <RuiTableRowExpander
        v-if="row.breakdown || (!hideBreakdown && isEvmNativeToken(row.asset))"
        :expanded="isExpanded(row.asset)"
        @click="expand(row)"
      />
    </template>
  </RuiDataTable>
</template>
