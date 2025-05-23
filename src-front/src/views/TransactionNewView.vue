<script setup>
  import { onBeforeMount, reactive, ref, nextTick } from 'vue';
  import { useRoute, useRouter } from 'vue-router';
  import { useI18n } from 'vue-i18n';
  import { DateTime } from 'luxon';

  import { Services } from '@/services/servicesConfig';
  import { processError } from '@/errors/errorHandlers';

  import TransactionTypeTabs from '@/components/transactions/TransactionTypeTabs.vue';
  import TransactionLabel from '@/components/transactions/TransactionLabel.vue';
  import TransactionAmount from '@/components/transactions/TransactionAmount.vue';
  import Category from '@/components/transactions/Category.vue';
  import AccountSelector from '@/components/transactions/AccountSelector.vue';
  import ExchangeRate from '@/components/transactions/ExchangeRate.vue';
  import TransactionDateTime from '@/components/transactions/TransactionDateTIme.vue';
  import { useUserStore } from '@/stores/user';

  const props = defineProps(['isEdit', 'returnUrl', 'accountId']);
  const router = useRouter();
  const route = useRoute();
  const { t } = useI18n();
  const userStore = useUserStore();

  let accounts = reactive([]);
  let currentAccount = reactive({});
  let targetAccount = reactive({});
  let categories = reactive([]);
  const showDeleteConfirmation = ref(false);
  const showSubmittedOverlay = ref(false);

  const amountComponent = ref(null);

  let transaction = reactive({
    label: '',
    amount: 0,
    categoryId: null,
    accountId: null,
    targetAccountId: null,
    targetAmount: null,
    isTransfer: false,
    isIncome: false,
    dateTime: null,
    notes: ''
  });
  const saveAsTemplate = ref(false);
  let filteredCategories = reactive([]);

  const itemType = ref('expense');
  transaction.isTransfer = itemType.value === 'transfer';

  const returnUrlName = ref('');

  function changeAccount({ accountType, accountId }) {
    if (accountType === 'src') {
      transaction.accountId = accountId;
    } else if (accountType === 'target') {
      transaction.targetAccountId = accountId;
    }
  }

  function amountChanged({ amountType, amount }) {
    if (transaction.isTransfer) {
      if (amountType === 'src') {
        transaction.amount = amount;
      } else {
        transaction.targetAmount = amount;
      }
    } else {
      transaction.amount = amount;
    }
  }

  onBeforeMount(async () => {
    if (props.returnUrl === 'accountDetails') {
      returnUrlName.value = 'accountDetails';
    } else {
      returnUrlName.value = 'transactions';
    }

    try {
      categories = await Services.categoriesService.getUserCategories();
      accounts.length = 0;
      accounts.push(...(await Services.accountsService.getUserAccounts({
        shouldUpdate: true, includeHidden: true
      })));
      userStore.transactionTemplates.splice(0, userStore.transactionTemplates.length,
        ...(await Services.transactionsService.getUserTemplates()));
      if (props.accountId) {
        currentAccount = accounts.find((item) => item.id === parseInt(props.accountId, 10));
      } else {
        currentAccount = accounts[0];
      }
      targetAccount = accounts[0];
      transaction.accountId = currentAccount.id;
      transaction.targetAccountId = targetAccount.id;

      if (props.isEdit) {
        let details = await Services.transactionsService.getTransactionDetails(route.params.id);
        Object.assign(transaction, details);
        itemType.value = transaction.isTransfer ? 'transfer' : transaction.isIncome ? 'income' : 'expense';

        // !!!!!!!
        // if we have an income part of transfer, we load and edit the expense part
        // it allows to simplify MUCH logic and UI
        if (transaction.isTransfer && transaction.isIncome) {
          details = await Services.transactionsService.getTransactionDetails(transaction.linkedTransactionId);
          transaction = Object.assign(transaction, details);
        }

        currentAccount = accounts.find((item) => item.id === transaction.accountId);

        if (transaction.linkedTransactionId) {
          await getLinkedTransaction(transaction);
        }
      }

      filterCategories();
    } catch (e) {
      await processError(e, router);
    }
  });

  async function getLinkedTransaction(transaction) {
    const linkedTransaction = await Services.transactionsService.getTransactionDetails(transaction.linkedTransactionId);
    transaction.targetAccountId = linkedTransaction.accountId;
    transaction.targetAmount = linkedTransaction.amount;
  }

  function filterCategories() {
    const isIncome = itemType.value === 'income';
    // filter categories by income/expense
    filteredCategories = categories.filter((item) => item.isIncome === isIncome);

    // if the current category is not in the filtered categories, set the first one from the filtered list
    if (!filteredCategories.some((item) => item.id === transaction.categoryId)) {
      transaction.categoryId = filteredCategories[0].id;
    }

    updateTransactionProperties(itemType.value);
  }

  function updateTransactionProperties(type) {
    if (type === 'transfer') {
      transaction.categoryId = null;
      transaction.isTransfer = true;
    } else {
      transaction.targetAccountId = null;
      transaction.targetAmount = null;
      transaction.isTransfer = false;
      transaction.categoryId = transaction.categoryId || filteredCategories.value[0].id;
      transaction.isIncome = itemType.value === 'income';
    }
  }

  function changeNotes($event) {
    transaction.notes = $event.target.value;
  }

  function changeItemType(type) {
    itemType.value = type;
    filterCategories();
  }

  function dateTimeChanged({ date, time }) {
    const dateTimeString = `${date}T${time}`;
    const localDateTime = DateTime.fromISO(dateTimeString, { zone: 'local' });
    transaction.dateTime = localDateTime.toUTC().toISO().replace('.000', '');
  }

  async function submitTransaction() {
    showSubmittedOverlay.value = true; // disable submit button
    if (saveAsTemplate.value) {
      transaction.isTemplate = true;
    } else {
      transaction.isTemplate = false;
    }

    try {
      if (props.isEdit) {
        await Services.transactionsService.updateTransaction(transaction);
      } else {
        await Services.transactionsService.addTransaction(transaction);
      }

      await router.push({
        name: returnUrlName.value,
        params: {
          id: props.accountId,
        },
      });
      showSubmittedOverlay.value = false; // enable submit button
    } catch (e) {
      await processError(e, router);
    }
  }

  function confirmDelete() {
    showDeleteConfirmation.value = true;
  }

  async function deleteTransaction() {
    try {
      await Services.transactionsService.deleteTransaction(transaction.id);
      await router.push({ name: 'transactions' });
    } catch (e) {
      await processError(e, router);
    } finally {
      showDeleteConfirmation.value = false;
    }
  }

  function updateLabel(value) {
    if (typeof value === 'string') {
      transaction.label = value;
    } else if (typeof value === 'object' && value !== null) {
      const selectedCategory = categories.find((item) => item.id === value.categoryId);
      transaction.label = value.label;
      transaction.categoryId = selectedCategory.id;
      transaction.isIncome = selectedCategory.isIncome;
      transaction.isTransfer = selectedCategory.isTransfer;
      itemType.value = selectedCategory.isIncome ? 'income' : 'expense';

      filterCategories();

      nextTick(() => {
        if (amountComponent.value?.amountInput) {
          const inputElement = amountComponent.value.amountInput;
          inputElement.focus();
          inputElement.select();
        }
      });
    }
  }
</script>

<template>
  <main>
    <div class="container">
      <div class="row">
        <div class="col">
          <form @submit.prevent="submitTransaction">
            <TransactionTypeTabs @type-changed="changeItemType" :is-edit="props.isEdit" :transaction="transaction"
                                 :item-type="itemType" />

            <TransactionLabel :transaction="transaction" @update:label="updateLabel" />

            <div class="mb-3">
              <input type="checkbox" id="saveAsTemplate" v-model="saveAsTemplate" class="form-check-input" />
              <label for="saveAsTemplate" class="form-label ms-2">{{ $t('message.saveAsTemplate') }}</label>
            </div>

            <AccountSelector @account-changed="changeAccount" :label="$t('message.account')" account-type="src"
                             :accountId="transaction.accountId" :accounts="accounts" />

            <TransactionAmount :label="t('message.amount')" ref="amountComponent" :amount-input="amountInput" type="src"
                               @amount-changed="amountChanged" :amount="transaction.amount"
                               :current-account="currentAccount" />

            <AccountSelector v-if="itemType === 'transfer'" @account-changed="changeAccount"
                             :label="$t('message.targetAccount')" account-type="target"
                             :accountId="transaction.targetAccountId"
                             :accounts="accounts" />

            <TransactionAmount v-if="itemType === 'transfer'" type="target" :label="t('message.amount')"
                               @amount-changed="amountChanged" :amount="transaction.targetAmount"
                               :current-account="targetAccount" />

            <ExchangeRate v-if="itemType === 'transfer'" :currency-src-code="currentAccount.currency.code"
                          :src-amount="transaction.amount" :currency-target-code="targetAccount.currency.code"
                          :target-amount="transaction.targetAmount" />

            <Category v-if="!transaction.isTransfer" :transaction="transaction" :categories="filteredCategories"
                      @update:categoryId="transaction.categoryId = $event" />

            <TransactionDateTime :transaction="transaction" :is-edit="isEdit" @date-time-changed="dateTimeChanged" />

            <div class="mb-3">
              <label for="notes" class="form-label">{{ $t('message.notes') }}</label>
              <textarea @keyup="changeNotes" class="form-control" id="notes" v-model="transaction.notes"
                        rows="3"></textarea>
            </div>

            <div class="flex-container">
              <button type="submit" class="btn btn-primary">{{ $t('buttons.submit') }}</button>
              <button @click.prevent="confirmDelete" class="btn btn-danger">{{ $t('buttons.delete') }}</button>
            </div>
          </form>

          <div v-if="showSubmittedOverlay" id="submitted-overlay" class="overlay"></div>

          <div v-if="showDeleteConfirmation" class="overlay"></div>

          <div v-if="showDeleteConfirmation" class="delete-confirmation">
            <p>Delete the transaction?</p>
            <div class="buttons-container">
              <button @click="deleteTransaction" class="btn btn-danger">{{ $t('buttons.yes') }}</button>
              <button @click="showDeleteConfirmation = false" class="btn btn-secondary">{{ $t('buttons.cancel') }}
              </button>
            </div>
          </div>
        </div>
      </div>
    </div>
  </main>
</template>

<style scoped>
  .flex-container {
    display: flex;
    flex-direction: row;
    justify-content: space-between;
  }

  .delete-confirmation {
    position: fixed;
    top: 50%;
    left: 50%;
    transform: translate(-50%, -50%);
    padding: 20px;
    background-color: white;
    border: 1px solid #ccc;
    border-radius: 5px;
    box-shadow: 0 2px 10px rgba(0, 0, 0, 0.1);
    z-index: 1000;
    text-align: center;
    width: auto;
  }

  .delete-confirmation p {
    margin-bottom: 20px;
    font-size: 16px;
    color: #333;
  }

  .buttons-container {
    display: flex;
    justify-content: center;
    gap: 10px;
  }

  .overlay {
    position: fixed;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    background-color: rgba(0, 0, 0, 0.5);
    z-index: 999;
  }
</style>