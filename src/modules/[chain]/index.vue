<script lang="ts" setup>
import MdEditor from 'md-editor-v3';
import PriceMarketChart from '@/components/charts/PriceMarketChart.vue';
import List from "./account/component.vue"

import { Icon } from '@iconify/vue';
import {
  useBlockchain,
  useFormatter,
  useTxDialog,
  useWalletStore,
  useStakingStore,
  useParamStore,
} from '@/stores';
import { onMounted, ref, watch } from 'vue';
import { useIndexModule, colorMap } from './indexStore';
import { computed } from '@vue/reactivity';

import CardStatisticsVertical from '@/components/CardStatisticsVertical.vue';
import ProposalListItem from '@/components/ProposalListItem.vue';
import ArrayObjectElement from '@/components/dynamic/ArrayObjectElement.vue'
import { bech32 } from 'bech32';
const props = defineProps(['chain']);

const blockchain = useBlockchain();
const store = useIndexModule();
const walletStore = useWalletStore();
const format = useFormatter();
const dialog = useTxDialog();
const stakingStore = useStakingStore();
const paramStore = useParamStore()
const coinInfo = computed(() => {
  return store.coinInfo;
});
const isEvmFormat = ref(true);
// Computed property to determine the button text
const addressFormat = computed(() => (isEvmFormat.value ? 'EVM' : 'Cosmos'));

// Method to toggle the address format
const toggleAddressFormat = () => {
  isEvmFormat.value = !isEvmFormat.value;
};


onMounted(() => {
  store.loadDashboard();
  walletStore.loadMyAsset();
  paramStore.handleAbciInfo()
  // if(!(coinInfo.value && coinInfo.value.name)) {
  // }
});
const ticker = computed(() => store.coinInfo.tickers[store.tickerIndex]);

const currName = ref("")
blockchain.$subscribe((m, s) => {
  if (s.chainName !== currName.value) {
    currName.value = s.chainName
    store.loadDashboard();
    walletStore.loadMyAsset();
    paramStore.handleAbciInfo()
  }
});
function shortName(name: string, id: string) {
  return name.toLowerCase().startsWith('ibc/') ||
    name.toLowerCase().startsWith('0x')
    ? id
    : name;
}

const comLinks = [
  {
    name: 'Website',
    icon: 'mdi-web',
    href: store.homepage,
  },
  {
    name: 'Twitter',
    icon: 'mdi-twitter',
    href: store.twitter,
  },
  {
    name: 'Telegram',
    icon: 'mdi-telegram',
    href: store.telegram,
  },
  {
    name: 'Github',
    icon: 'mdi-github',
    href: store.github,
  },
];

// wallet box
const change = computed(() => {
  const token = walletStore.balanceOfStakingToken;
  return token ? format.priceChanges(token.denom) : 0;
});
const color = computed(() => {
  switch (true) {
    case change.value > 0:
      return 'text-green-600';
    case change.value === 0:
      return 'text-grey-500';
    case change.value < 0:
      return 'text-red-600';
  }
});

function updateState() {
  walletStore.loadMyAsset()
}

function trustColor(v: string) {
  return `text-${colorMap(v)}`
}

const quantity = ref(100)
const qty = computed({
  get: () => {
    return parseFloat(quantity.value.toFixed(6))
  },
  set: val => {
    quantity.value = val
  }
})
const amount = computed({
  get: () => {
    return quantity.value * ticker.value.converted_last.usd || 0
  },
  set: val => {
    quantity.value = val / ticker.value.converted_last.usd || 0
  }
})

function cosmosToEvmAddress(cosmosAddress: string): string {
  // 检查 bech32 是否已正确导入
  if (!cosmosAddress) {
    return ''
  }

  // 解码 Bech32 地址
  const decoded = bech32.decode(cosmosAddress);
  const pubkeyHash = new Uint8Array(bech32.fromWords(decoded.words));

  // 将公钥哈希转换为 EVM 地址
  const evmAddress = '0x' + Array.from(pubkeyHash)
    .map(byte => byte.toString(16).padStart(2, '0'))
    .join('');

  return evmAddress;
}
const currentAddress2 = computed(() => {
  return isEvmFormat.value
    ? cosmosToEvmAddress(walletStore.currentAddress)
    : walletStore.currentAddress;
})

// Watch for changes in currentAddress2
watch(() => walletStore.currentAddress, async (newAddress, oldAddress) => {
    if (newAddress !== oldAddress) {
        await walletStore.loadMyAsset();
    }
});
</script>

<template>
  <div>
    <div class="bg-base-100 rounded mt-4 shadow">
      <div class="flex justify-between px-4 pt-4 pb-2 text-lg font-semibold text-main">
        <div class="flex flex-col gap-2">
          <div class="text-lg font-medium">
            Current Address
          </div>
          <div class="flex sm:flex-row flex-col w-full gap-4" v-if="walletStore.currentAddress">
            <span class="text-[#000014B2] truncate text-xs sm:text-sm underline underline-offset-1">
              {{ currentAddress2 || 'Not Connected' }}
            </span>
            <div
              class="border-1 rounded-sm flex justify-center max-w-fit items-center px-2 gap-1 cursor-pointer leading-3 text-[10px] border-black"
              @click="toggleAddressFormat">
              {{ addressFormat }}
              <img src="../../assets/page/switch.svg" />
            </div>
          </div>
          <div class="flex" v-else>
            <label v-if="!walletStore?.currentAddress" for="PingConnectWallet"
              class="border-[#000014] py-0 px-3 border-1 cursor-pointer font-medium">
              <span class="ml-1 block">Connect Wallet</span>
            </label>
          </div>
        </div>
        <RouterLink v-if="walletStore.currentAddress"
          class="float-right text-sm cursor-pointert link link-primary no-underline font-medium"
          :to="`/${chain}/account/${walletStore.currentAddress}`">{{ $t('index.more') }}</RouterLink>
      </div>
      <div class="grid grid-cols-1 md:!grid-cols-5 auto-cols-auto gap-4 px-4 pb-6 mt-3">
        <div class="bg-[#E2E6FF] dark:bg-[#373f59] rounded-sm px-4 py-3">
          <div class="text-sm mb-1">{{ $t('account.balance') }}</div>
          <div class="text-lg font-semibold text-main">
            {{ walletStore.currentAddress ? (walletStore.balanceOfStakingToken && walletStore.balanceOfStakingToken.amount !== '0' ? format.formatToken(walletStore.balanceOfStakingToken) : '--') : '--' }}
          </div>
        </div>
        <div class="bg-[#FFF4DE] dark:bg-[#373f59] rounded-sm px-4 py-3">
          <div class="text-sm mb-1">{{ $t('module.staking') }}</div>
          <div class="text-lg font-semibold text-main">
            {{ walletStore.currentAddress ? format.formatToken(walletStore.stakingAmount) : '--' }}
          </div>
        </div>
        <div class="bg-[#DCFCE7] dark:bg-[#373f59] rounded-sm px-4 py-3">
          <div class="text-sm mb-1">{{ $t('index.reward') }}</div>
          <div class="text-lg font-semibold text-main">
            {{ walletStore.currentAddress ? format.formatToken(walletStore.rewardAmount) : '--' }}
          </div>
        </div>
        <div class="bg-[#FFE2E5] dark:bg-[#373f59] rounded-sm px-4 py-3">
          <div class="text-sm mb-1">{{ $t('index.unbonding') }}</div>
          <div class="text-lg font-semibold text-main">
            {{ walletStore.currentAddress ? format.formatToken(walletStore.unbondingAmount) : '--' }}
          </div>
        </div>
        <div class="bg-[#fae4e4] dark:bg-[#373f59] rounded-sm px-4 py-3">
          <div class="text-sm mb-1">Current Inflation</div>
          <div class="text-lg font-semibold text-main">
            {{ store.stats[4].stats }}
          </div>
        </div>
      </div>
    </div>

    <!-- <div class="bg-base-100 mt-6 p-4">
      <div class="text-lg font-medium">
        My Delegations
      </div>
      <div v-if="walletStore.delegations.length > 0" class="pb-4 overflow-auto mt-2">
        <table class="table table-compact w-full table-zebra">
          <thead>
            <tr>
              <th>{{ $t('account.validator') }}</th>
              <th>{{ $t('account.delegations') }}</th>
              <th>{{ $t('account.rewards') }}</th>
              <th>{{ $t('staking.actions') }}</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="(item, index) in walletStore.delegations" :key="index">
              <td>
                <RouterLink class="link link-primary no-underline"
                  :to="`/${chain}/staking/${item?.delegation?.validator_address}`">
                  {{
                    format.validatorFromBech32(
                  item?.delegation?.validator_address
                  )
                  }}
                </RouterLink>
              </td>
              <td>{{ format.formatToken(item?.balance) }}</td>
              <td>
                {{
                  format.formatTokens(
                    walletStore?.rewards?.rewards?.find(
                (el) =>
                el?.validator_address ===
                item?.delegation?.validator_address
                )?.reward)
                }}
              </td>
              <td>
                <div>
                  <label for="delegate" class="btn !btn-xs !btn-primary btn-ghost rounded-sm mr-2"
                    @click="dialog.open('delegate', { validator_address: item.delegation.validator_address }, updateState)">
                    {{ $t('account.btn_delegate') }}
                  </label>
                  <label for="withdraw" class="btn !btn-xs !btn-primary btn-ghost rounded-sm mr-2"
                    @click="dialog.open('withdraw', { validator_address: item.delegation.validator_address }, updateState)">
                    {{ $t('index.btn_withdraw_reward') }}
                  </label>
                  <label for="unbond" class="btn !btn-xs !btn-primary btn-ghost rounded-sm mr-2"
                    @click="dialog.open('unbond', { validator_address: item.delegation.validator_address }, updateState)">
                    {{ $t('account.btn_unbond') }}
                  </label>
                </div>
              </td>
            </tr>
          </tbody>
        </table>
      </div>

    </div> -->
    <List :address="walletStore.currentAddress" />
  </div>
</template>

<route>
  {
    meta: {
      i18n: 'dashboard',
      order: 1,
      icon: 'wallet.svg'
    }
  }
</route>
