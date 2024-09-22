<template>
  <v-row justify="center">
    <v-dialog v-model="offlineBillsDialog" max-width="900px">
      <v-card>
        <v-card-title>
          <span class="headline primary--text">{{
            __('Select Offline Bill')
          }}</span>
        </v-card-title>
        <v-card-text class="pa-0">
          <v-container>
            <v-row class="mb-4">
              <v-col cols="3">
                <v-text-field
                  color="primary"
                  :label="frappe._('Invoice ID *')"
                  background-color="white"
                  hide-details
                  v-model="invoice_name"
                  dense
                  clearable
                  @keydown.enter="get_fs_offline_invoices"
              ></v-text-field>
              </v-col>
              <v-col cols="3">
                <v-text-field
                  color="primary"
                  :label="frappe._('Customer Name *')"
                  background-color="white"
                  hide-details
                  v-model="customer_name"
                  dense
                  clearable
                  @keydown.enter="get_fs_offline_invoices"
                ></v-text-field>
              </v-col>
              <v-col cols="3">
                <v-text-field
                  color="primary"
                  :label="frappe._('Full FS Account No.*')"
                  background-color="white"
                  hide-details
                  v-model="custom_fs_account_number"
                  dense
                  clearable
                  @keydown.enter="get_fs_offline_invoices"
                ></v-text-field>
              </v-col>
             </v-row>
            <v-row no-gutters>
              <v-col cols="12" class="pa-1" v-if="dialog_data">
                <template>
                  <v-data-table
                    :headers="headers"
                    :items="dialog_data"
                    item-key="name"
                    class="elevation-1"
                    :single-select="singleSelect"
                    show-select
                    v-model="selected"
                  >
                    <template v-slot:item.posting_time="{ item }">
                      {{ item.posting_time.split('.')[0] }}
                    </template>
                    <template v-slot:item.grand_total="{ item }">
                      {{ currencySymbol(item.currency) }}
                      {{ formtCurrency(item.grand_total) }}
                    </template>
                  </v-data-table>
                </template>
              </v-col>
            </v-row>
          </v-container>
        </v-card-text>
        <v-card-actions>
          <v-spacer></v-spacer>
            <v-btn
              text
              class="ml-2"
              color="primary"
              dark
              @click="get_fs_offline_invoices"
            >{{ __('Search') }}
            </v-btn>          
          <v-btn color="error mx-2" dark @click="close_dialog">Close</v-btn>
          <v-btn
            v-if="selected.length"
            color="success"
            dark
            @click="submit_dialog"
            >{{ __('Select') }}</v-btn
          >
        </v-card-actions>
      </v-card>
    </v-dialog>
  </v-row>
</template>

<script>
import { evntBus } from '../../bus';
import format from '../../format';
export default {
  mixins: [format],
  data: () => ({
    offlineBillsDialog: false,
    singleSelect: true,
    selected: [],
    dialog_data: '',
    company: '',
    invoice_name: '', // for return search
    customer_name: '', // for return search
    custom_fs_account_number: '', // for return search
    headers: [
      {
        text: __('Customer'),
        value: 'customer_name',
        align: 'start',
        sortable: true,
      },
      {
        text: __('Date'),
        align: 'start',
        sortable: true,
        value: 'posting_date',
      },
      {
        text: __('Time'),
        align: 'start',
        sortable: true,
        value: 'posting_time',
      },
      {
        text: __('Invoice'),
        value: 'name',
        align: 'start',
        sortable: true,
      },
      {
        text: __('Amount'),
        value: 'grand_total',
        align: 'end',
        sortable: false,
      },
    ],
  }),
  watch: {},
  methods: {
    close_dialog() {
      this.offlineBillsDialog = false;
    },

    get_fs_offline_invoices() {
      const vm = this;
      frappe.call({
        method: 'posawesome.posawesome.api.posapp.get_fs_offline_invoices',
        args: {
          invoice_name: vm.invoice_name,
          customer_name: vm.customer_name,
          custom_fs_account_number: vm.custom_fs_account_number,
          company: vm.company
        },
        async: false,
        callback: function (r) {
          if (r.message) {
            if (r.message instanceof Array) { // check whether there is a result or an error message
              vm.dialog_data = r.message;
            }
            else {
              evntBus.$emit('show_mesage', {
                text: r.message,
                color: 'warning',
              });
            };
          };
        },
      });
    },

    submit_dialog() {
      if (this.selected.length > 0) {
        evntBus.$emit('load_invoice', this.selected[0]);
        this.offlineBillsDialog = false;
      }
    },
  },
  created: function () {
    evntBus.$on('open_offlineBill_drafts', (data) => {
      this.selected = []; // to reset old selections
      this.offlineBillsDialog = true;
      this.company = data;
      this.invoice_name = '';
      this.customer_name = '';
      this.custom_fs_account_number = '';
      this.dialog_data = '';
    });
  },
};
</script>
