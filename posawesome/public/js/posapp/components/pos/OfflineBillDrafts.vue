<template>
  <v-row justify="center">
    <v-dialog v-model="draftsDialog" max-width="900px">
      <!-- <template v-slot:activator="{ on, attrs }">
        <v-btn color="primary" dark v-bind="attrs" v-on="on">Open Dialog</v-btn>
      </template>-->
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
                ></v-text-field>
              </v-col>
             </v-row>
            <v-row no-gutters>
              <v-col cols="12" class="pa-1">
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
          <v-btn color="error" dark @click="close_dialog">Close</v-btn>
          <v-btn color="success" dark @click="submit_dialog">Select</v-btn>
        </v-card-actions>
      </v-card>
    </v-dialog>
  </v-row>
</template>

<script>
import { evntBus } from '../../bus';
import format from '../../format';
export default {
  // props: ["draftsDialog"],
  mixins: [format],
  data: () => ({
    draftsDialog: false,
    singleSelect: true,
    selected: [],
    dialog_data: {},
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
      this.draftsDialog = false;
    },

    search_invoices_by_enter(e) {
      if (e.keyCode === 13) {
        this.get_fs_offline_invoices();
      }
    },

    get_fs_offline_invoices() {
      console.log(invoice_name, customer_name, custom_fs_account_number);
      const vm = this;
      frappe.call({
        method: 'posawesome.posawesome.api.posapp.get_fs_offline_invoices',
        args: {
          invoice_name: vm.invoice_name,
          customer_name: vm.customer_name,
          custom_fs_account_number: vm.custom_fs_account_number,
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
            }
          }
        },
      });
    },

    submit_dialog() {
      if (this.selected.length > 0) {
        evntBus.$emit('load_invoice', this.selected[0]);
        this.draftsDialog = false;
      }
    },
  },
  created: function () {
    document.addEventListener("keydown", this.search_invoices_by_enter.bind(this));
    evntBus.$on('open_offlineBill_drafts', () => {
      this.selected = []; // to reset old selections
      this.draftsDialog = true;
      //this.dialog_data = data;
    });
  },
  destroyed() {
    document.removeEventListener("keydown", this.search_invoices_by_enter);
  }
};
</script>
