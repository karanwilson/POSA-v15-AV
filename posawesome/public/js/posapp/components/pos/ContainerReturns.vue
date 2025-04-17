<template>
  <v-row justify="center">
    <v-dialog v-model="invoicesDialog" max-width="1000px" min-width="1000px">
      <v-card>
        <v-card-title>
          <span class="headline primary--text">{{
            __('Container Returns')
          }}</span>
        </v-card-title>
        <v-container>
          <v-row class="mb-4">
            <v-col cols="3">
              <v-text-field
                color="primary"
                :label="frappe._('Full PT Account No.*')"
                background-color="white"
                hide-details
                v-model="custom_fs_account_number"
                dense
                clearable
                @keydown.enter="search_return_item"
              ></v-text-field>
            </v-col>
          </v-row>
          <v-row class="mb-4">
            <v-col align="center" cols="4">
              <v-checkbox
                dense
                :label="frappe._('0011 - 1L AV MILK GLASS BOTTLE')"
                v-model="returnItem_0011"
                hide-details
                class="shrink mr-2 mt-0"
              ></v-checkbox>
            </v-col>
            <v-col cols="1">
              <v-text-field
                color="secondary"
                :label="frappe._('Qty')"
                background-color="white"
                hide-details
                v-model="returnQty_0011"
                dense
                clearable
                @keydown.enter="search_return_item"
              ></v-text-field>
            </v-col>
            <v-col align="center" cols="4">
              <v-checkbox
                dense
                :label="frappe._('0055- 0.5L AV MILK GLASS BOTTLE')"
                v-model="returnItem_0055"
                hide-details
                class="shrink mr-2 mt-0"
              ></v-checkbox>
            </v-col>
            <v-col cols="1">
              <v-text-field
                color="secondary"
                :label="frappe._('Qty')"
                background-color="white"
                hide-details
                v-model="returnQty_0055"
                dense
                clearable
                @keydown.enter="search_return_item"
              ></v-text-field>
            </v-col>
            <v-col align="center" cols="4">
              <v-checkbox
                dense
                :label="frappe._('10060 - OLIVE OIL BOTTLE 500ML')"
                v-model="returnItem_10060"
                hide-details
                class="shrink mr-2 mt-0"
              ></v-checkbox>
            </v-col>
            <v-col cols="1">
              <v-text-field
                color="secondary"
                :label="frappe._('Qty')"
                background-color="white"
                hide-details
                v-model="returnQty_10060"
                dense
                clearable
                @keydown.enter="search_return_item"
              ></v-text-field>
            </v-col>
          </v-row>
          <v-row>
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
                  <template v-slot:item.grand_total="{ item }">
                    {{ currencySymbol(item.currency) }}
                    {{ formtCurrency(item.grand_total) }}</template
                  >
                </v-data-table>
              </template>
            </v-col>
          </v-row>
        </v-container>
        <v-card-actions class="mt-4">
          <v-spacer></v-spacer>
          <v-btn color="error mx-2" dark @click="close_dialog">Close</v-btn>
          <v-btn
            v-if="returnItem_0011 || returnItem_0055 || returnItem_10060"
            color="success"
            dark
            @click="create_return_invoice_for_container"
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
    invoicesDialog: false,
    singleSelect: true,
    selected: [],
    dialog_data: '',
    company: '',
    invoice_name: '',
    customer_name: '', // for return search
    custom_fs_account_number: '', // for return search
    item_code: '', // for return search
    //pos_profile: '', // for matching bottle returns
    //returnableContainers: ["0011", "0055", "10060"],
    //returnContainer: '',
    returnItem_0011: 0,
    returnItem_0055: 0,
    returnItem_10060: 0,
    returnQty_0011: 1,
    returnQty_0055: 1,
    returnQty_10060: 1,
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
      this.invoicesDialog = false;
    },
    search_return_item(e) {
      if (e.keyCode === 13) {
        this.create_return_invoice_for_container();
      }
    },
    create_return_invoice_for_container() {
      const vm = this;
      frappe.call({
        method: 'posawesome.posawesome.api.posapp.create_return_invoice_for_container',
        args: {
          custom_fs_account_number: vm.custom_fs_account_number,
          item_code: vm.item_code,
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
      if (returnItem_0011 || returnItem_0055 || returnItem_10060) {

      }
      if (this.selected.length > 0) {
        const return_doc = this.selected[0];
        const invoice_doc = {};
        const items = [];
        return_doc.items.forEach((item) => {
          const new_item = { ...item };
          new_item.qty = item.qty * -1;
          new_item.stock_qty = item.stock_qty * -1;
          new_item.amount = item.amount * -1;
          new_item.sales_invoice_item = item.name; // needed during returns validation in the new update ERPNext v14.83.0, Frappe v14.94.1
          items.push(new_item);
        });
        invoice_doc.items = items;
        invoice_doc.is_return = 1;
        invoice_doc.return_against = return_doc.name;
        invoice_doc.customer = return_doc.customer;
        const data = { invoice_doc, return_doc };
        evntBus.$emit('load_return_invoice', data);
        this.invoicesDialog = false;
      }
    },
  },
  mounted() {
    evntBus.$on("register_pos_profile", (data) => {
      this.pos_profile = data.pos_profile;
    });
  },
  created: function () {
    evntBus.$on('open_container_returns', (data) => {
      this.invoicesDialog = true;
      this.company = data;
      this.invoice_name = '';
      this.customer_name = '';
      this.custom_fs_account_number = '';
      this.item_code = '';
      //this.returnContainer = '',
      //this.returnQty = 1,
      this.returnItem_0011 = 0;
      this.returnItem_0055 = 0;
      this.returnItem_10060 = 0;
      this.returnQty_0011 = 1;
      this.returnQty_0055 = 1;
      returnQty_10060: 1;
      this.dialog_data = '';
      this.selected = [];
    });
  },
};
</script>
