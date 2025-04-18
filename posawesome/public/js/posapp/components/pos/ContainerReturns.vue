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
                @keydown.enter="search_customer"
              ></v-text-field>
            </v-col>
            <v-col cols="3">
              <v-text-field
                color="primary"
                :label="frappe._('Participant')"
                background-color="white"
                hide-details
                v-model="customer.customer_name"
                readonly
                dense
                clearable
              ></v-text-field>
            </v-col>
          </v-row>
          <v-row>
            <v-col cols="12" class="pa-1" v-if="dialog_data">
              <template>
                <v-data-table
                  :headers="headers"
                  :items="dialog_data"
                  item-key="item_code"
                  class="elevation-1"
                  show-select
                  v-model="selected"
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
            v-if="dialog_data"
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
    invoicesDialog: false,
    singleSelect: true,
    selected: [],
    dialog_data: '',
    company: '',
    items: [],
    //invoice_name: '',
    //customer_name: '', // for return search
    customer: '',
    custom_fs_account_number: '', // for return search
    //item_code: '', // for return search
    pos_profile: '', // for matching bottle returns
    posting_date: frappe.datetime.nowdate(),
    headers: [
      {
        text: __('Name'),
        value: 'item_name',
        align: 'start',
        sortable: true,
      },
      {
        text: __('Code'),
        value: 'item_code',
        align: 'start',
        sortable: true,
      },
      {
        text: __('Qty'),
        value: 'qty',
        align: 'start',
        sortable: true,
      },
      {
        text: __('Rate'),
        value: 'rate',
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
    search_customer() {
      const vm = this;
      frappe.call({
        method: 'posawesome.posawesome.api.posapp.get_customer',
        args: {
          custom_fs_account_number: vm.custom_fs_account_number,
        },
        async: false,
        callback: function (r) {
          if (r.message) {
            if (r.message.error) {
              evntBus.$emit('show_mesage', {
                text: r.message,
                color: 'warning',
              });
            }
            else {
              vm.customer = r.message[0]; // value comes as an array of dictionary
              //console.log("r.message: ", r.message);
            }
          }
        },
      });
    },

    get_items() {
      if (!this.pos_profile) {
        console.error("No POS Profile");
        return;
      }
      const vm = this;
      let gr = "Storage Containers";
      frappe.call({
        method: "posawesome.posawesome.api.posapp.get_items",
        args: {
          pos_profile: vm.pos_profile,
          item_group: gr,
        },
        callback: function (r) {
          if (r.message) {
            vm.dialog_data = r.message;
            vm.dialog_data.forEach((item) => {
              item.qty = 1;
            })
          }
        },
      });
    },

    get_invoice_items() {
      const items_list = [];
      //this.items.forEach((item) => {});
      for (const item of this.selected) {
        const new_item = {
          //sales_invoice_item: item.sales_invoice_item, // needed during returns validation in the new update ERPNext v14.83.0, Frappe v14.94.1
          item_code: item.item_code,
          item_name: item.item_name,
          qty: flt(item.qty),
          rate: flt(item.rate),
          uom: item.stock_uom,
          amount: flt(item.qty) * flt(item.rate),
        };
        items_list.push(new_item);
      }
      return items_list;
    },

    subtotal(items) {
      let sum = 0;
      items.forEach((item) => {
        sum += flt(item.qty) * flt(item.rate);
      });
      return this.flt(sum, this.currency_precision, undefined, this.rounding_method);
    },

    get_invoice_doc() {
      let doc = {};
      doc.doctype = "Sales Invoice";
      doc.is_pos = 1;
      doc.ignore_pricing_rule = 1;
      doc.company = doc.company || this.pos_profile.company;
      doc.pos_profile = doc.pos_profile || this.pos_profile.name;
      doc.items = this.get_invoice_items();
      doc.total = this.subtotal(doc.items);
      doc.grand_total = doc.total;
      doc.discount_amount = 0;
      doc.additional_discount_percentage = 0
      doc.payments = '';
      doc.taxes = [];
      doc.posa_delivery_charges_rate = this.delivery_charges_rate || 0;
      doc.posting_date = this.posting_date;
      return doc;
    },

    submit_dialog() {
      if (this.selected.length > 0) {
        //console.log("this.selected: ", this.selected);
        const return_doc = this.get_invoice_doc();
        const invoice_doc = {};
        const items = [];
        //console.log("return_doc: ", return_doc);
        return_doc.items.forEach((item) => {
          const new_item = { ...item };
          new_item.qty = item.qty * -1;
          new_item.stock_qty = item.qty * -1;
          new_item.amount = item.amount * -1;
          new_item.sales_invoice_item = item.name; // needed during returns validation in the new update ERPNext v14.83.0, Frappe v14.94.1
          items.push(new_item);
        });
        invoice_doc.items = items;
        invoice_doc.is_return = 1;
        invoice_doc.return_against = "";
        //invoice_doc.container_return = 1; // needed at posapp.py update_invoice as return_against is not set for container returns
        evntBus.$emit('container_return', true);
        //console.log('invoice_doc: ', invoice_doc);
        if (this.customer) {
          invoice_doc.customer = this.customer.name;
          const data = { invoice_doc, return_doc };
          evntBus.$emit('load_return_invoice', data);
          this.invoicesDialog = false;
        }
        else {
          evntBus.$emit('show_mesage', {
            text: "Please enter the PT Account Number",
            color: 'error',
          });
        }
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
      this.get_items();
      this.customer = '';
      this.custom_fs_account_number = '';
      this.dialog_data = '';
      this.selected = [];
    });
  },
};
</script>
