<template>
    <div>
        <div class="row">
            <div class="col-12 mb-4 pr-0 d-flex justify-content-between">
                <h4 class="mb-0">{{ $t('title') }}</h4>
                <div>
                    <button class="btn btn-sm btn-outline-dark"
                            :class="{ 'mr-3': !isStorageLocal }"
                            @click="createNewInvoice">{{ $t('new_invoice') }}
                    </button>
                    <b-dropdown variant="link" size="sm" no-caret right v-if="isStorageLocal">
                        <template slot="button-content">
                            <i class="material-icons">more_vert</i>
                        </template>
                        <b-dropdown-item @click="exportJson">{{ $t('export') }}</b-dropdown-item>
                        <b-dropdown-item @click="openImportModal">{{ $t('import') }}</b-dropdown-item>
                    </b-dropdown>
                </div>
            </div>
        </div>
        <div class="row mb-4">
            <div class="col-12 col-md-6 col-xl-3 mb-3 mb-xl-0">
                <div class="card h-100 dp--02">
                    <div class="card-body">
                        <small class="text-muted d-block mb-2">{{ $t('stats_amount_invoiced_this_year') }}</small>
                        <div class="invoice-stats__value">
                            <span v-if="amountInvoicedThisYear.length === 0">0</span>
                            <div v-else class="invoice-stats__currencies">
                                <div v-for="amount in amountInvoicedThisYear" :key="amount.currency">
                                    {{ amount.currency }} {{ amount.value | currency }}
                                </div>
                            </div>
                        </div>
                    </div>
                </div>
            </div>
            <div class="col-12 col-md-6 col-xl-3 mb-3 mb-xl-0">
                <div class="card h-100 dp--02">
                    <div class="card-body">
                        <small class="text-muted d-block mb-2">{{ $t('stats_invoices_created') }}</small>
                        <div class="invoice-stats__value">{{ invoicesCreatedCount }}</div>
                    </div>
                </div>
            </div>
            <div class="col-12 col-md-6 col-xl-3 mb-3 mb-md-0">
                <div class="card h-100 dp--02">
                    <div class="card-body">
                        <small class="text-muted d-block mb-2">{{ $t('stats_invoices_pending') }}</small>
                        <div class="invoice-stats__value">{{ pendingInvoicesCount }}</div>
                    </div>
                </div>
            </div>
            <div class="col-12 col-md-6 col-xl-3">
                <div class="card h-100 dp--02">
                    <div class="card-body">
                        <small class="text-muted d-block mb-2">{{ $t('stats_invoices_paid') }}</small>
                        <div class="invoice-stats__value">{{ paidInvoicesCount }}</div>
                    </div>
                </div>
            </div>
        </div>
        <div class="row">
            <div class="col-12">
                <InvoicesList/>
            </div>
        </div>
    </div>
</template>

<script>
import { BDropdown, BDropdownItem } from 'bootstrap-vue';
import { mapGetters } from 'vuex';
import InvoicesList from '@/components/invoices/InvoicesList';
import config from '@/config/app.config';
import { formatCurrency } from '@/filters/currency.filter';
import dayjs from 'dayjs';

export default {
  name: 'invoices',
  i18nOptions: { namespaces: 'invoices' },
  components: {
    InvoicesList,
    BDropdown,
    BDropdownItem,
  },
  filters: {
    currency: formatCurrency,
  },
  computed: {
    ...mapGetters({
      team: 'teams/team',
      invoices: 'invoices/all',
    }),
    isStorageLocal() {
      return config.storageType === 'local';
    },
    amountInvoicedThisYear() {
      const currentYear = dayjs().year();
      const yearlyTotalsByCurrency = this.invoices.reduce((totals, invoice) => {
        const issuedYear = dayjs(invoice.issued_at, 'YYYY-MM-DD').year();
        const isInCurrentYear = issuedYear === currentYear;
        const isInvoiced = !['draft', 'cancelled'].includes(invoice.status);

        if (!isInCurrentYear || !isInvoiced) {
          return totals;
        }

        const currency = invoice.currency || this.team.currency || 'USD';
        if (!Object.prototype.hasOwnProperty.call(totals, currency)) {
          totals[currency] = 0;
        }
        totals[currency] += invoice.total;
        return totals;
      }, {});

      return Object.keys(yearlyTotalsByCurrency)
        .sort()
        .map(currency => ({
          currency,
          value: yearlyTotalsByCurrency[currency],
        }));
    },
    invoicesCreatedCount() {
      return this.invoices.length;
    },
    pendingInvoicesCount() {
      return this.invoices.filter(invoice => !['paid', 'cancelled'].includes(invoice.status))
        .length;
    },
    paidInvoicesCount() {
      return this.invoices.filter(invoice => invoice.status === 'paid')
        .length;
    },
  },
  methods: {
    createNewInvoice() {
      this.$store.dispatch('invoices/createNewInvoice')
        .then(id => this.$router.push({ name: 'invoice', params: { id } }));
    },
    exportJson() {
      this.$store.dispatch('data/exportJson');
    },
    openImportModal() {
      this.$store.commit('data/isImportModalOpen', true);
    },
  },
};
</script>

<style lang="scss" scoped>
.invoice-stats__value {
    font-size: 1.25rem;
    font-weight: 600;
    line-height: 1.4;
}

.invoice-stats__currencies {
    display: flex;
    flex-direction: column;
    gap: 0.2rem;
}
</style>
