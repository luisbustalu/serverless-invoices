<template>
    <div class="row" v-if="invoice">
        <div class="col-12 mb-4 d-flex justify-content-between align-items-start">
            <router-link class="btn btn-sm btn-light btn--icon-left"
                         :to="{name: 'invoices'}">
                <i class="material-icons">arrow_back</i>
                <span class="d-inline-block">{{ $t('back') }}</span>
            </router-link>
            <div class="d-flex align-items-center">
                <AppSelect :value="getStatusObj"
                           class="mb-0 mr-2 text-capitalize multiselect--capitalize"
                           :options="invoiceStatuses"
                           label-field="name"
                           @input="updateProp({status: $event.value})"/>
                <button class="btn btn-outline-dark"
                        v-if="invoice.status === 'draft'"
                        @click="bookInvoice">{{ $t('book') }}
                </button>
                <b-dropdown variant="link" no-caret right>
                    <template slot="button-content">
                        <i class="material-icons">more_vert</i>
                    </template>
                    <b-dropdown-group :header="$t('design_and_layout')">
                        <b-dropdown-item-button @click="toggleCompact">
                            {{ invoice.is_compact ? $t('comfortable') : $t('compact') }}
                        </b-dropdown-item-button>
                        <b-dropdown-item-button @click="openCustomizationsModal">
                            {{ $t('customize') }}
                        </b-dropdown-item-button>
                    </b-dropdown-group>
                    <b-dropdown-divider/>
                    <b-dropdown-item-button @click="print">{{ $t('download_pdf') }}</b-dropdown-item-button>
                    <b-dropdown-item-button @click="cloneInvoice">{{ $t('clone_as_new') }}</b-dropdown-item-button>
                    <b-dropdown-item-button @click="deleteInvoice">{{ $t('delete') }}</b-dropdown-item-button>
                </b-dropdown>
            </div>
        </div>
    </div>
</template>

<script>
import { mapGetters, mapState } from 'vuex';
import NotificationService from '@/services/notification.service';
import {
  BDropdown,
  BDropdownDivider,
  BDropdownGroup,
  BDropdownItemButton,
} from 'bootstrap-vue';
import AppSelect from '@/components/form/AppSelect';

export default {
  i18nOptions: {
    namespaces: ['invoice-controls', 'statuses'],
  },
  components: {
    BDropdown,
    BDropdownDivider,
    BDropdownItemButton,
    BDropdownGroup,

    AppSelect,
  },
  computed: {
    ...mapState({
      theme: state => state.themes.theme,
    }),
    ...mapGetters({
      invoice: 'invoices/invoice',
    }),
    getStatusObj() {
      return this.invoiceStatuses
        .find(obj => obj.value === this.invoice.status);
    },
    invoiceStatuses() {
      return [{
        value: 'draft',
        name: this.$t('statuses.draft'),
      }, {
        value: 'booked',
        name: this.$t('statuses.booked'),
      }, {
        value: 'sent',
        name: this.$t('statuses.sent'),
      }, {
        value: 'paid',
        name: this.$t('statuses.paid'),
      }, {
        value: 'cancelled',
        name: this.$t('statuses.cancelled'),
      }];
    },
  },
  methods: {
    async deleteInvoice() {
      const confirmed = await this.$bvModal.msgBoxConfirm(`${this.$t('delete_modal.title')} ${this.invoice.number}?`, {
        okTitle: this.$t('delete_modal.ok_title'),
        okVariant: 'danger',
        cancelTitle: this.$t('delete_modal.cancel_title'),
        cancelVariant: 'btn-link',
        contentClass: 'bg-base dp--24',
      });
      if (confirmed) {
        await this.$store.dispatch('invoices/deleteInvoice', this.invoice);
        NotificationService.success('Deleted');
        this.$router.push({
          name: 'invoices',
        });
      }
    },
    bookInvoice() {
      this.$store.dispatch('invoices/bookInvoice');
    },
    updateProp(props) {
      this.$store.dispatch('invoices/updateInvoice', {
        props,
        invoiceId: this.invoice.id,
      });
    },
    toggleCompact() {
      this.updateProp({ is_compact: !this.invoice.is_compact });
    },
    openCustomizationsModal() {
      this.$store.commit('invoices/isCustomizationsModalOpen', true);
    },
    print() {
      const previousTheme = this.theme || 'light';
      const restoreTheme = () => {
        this.$store.commit('themes/theme', previousTheme);
        document.documentElement.setAttribute('data-theme', previousTheme);
        window.removeEventListener('afterprint', restoreTheme);
      };

      if (previousTheme === 'dark') {
        this.$store.commit('themes/theme', 'light');
        document.documentElement.setAttribute('data-theme', 'light');
        window.addEventListener('afterprint', restoreTheme);
      }

      window.print();

      // Some browsers may not fire afterprint reliably.
      if (previousTheme === 'dark') {
        setTimeout(restoreTheme, 200);
      }
    },
    async cloneInvoice() {
      const id = await this.$store.dispatch('invoices/cloneInvoice', this.invoice.id);
      NotificationService.success('Cloned');
      this.$router.push({
        name: 'invoice',
        params: { id },
      });
    },
  },
};
</script>
