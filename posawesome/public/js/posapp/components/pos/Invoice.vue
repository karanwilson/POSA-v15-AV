<template>
  <div>
    <v-dialog v-model="cancel_dialog" max-width="330">
      <v-card>
        <v-card-title class="text-h5">
          <span class="headline primary--text">{{
            __("Cancel Current Invoice ?")
          }}</span>
        </v-card-title>
        <v-card-actions>
          <v-spacer></v-spacer>
          <v-btn color="error" @click="cancel_invoice">
            {{ __("Cancel") }}
          </v-btn>
          <v-btn color="warning" @click="cancel_dialog = false">
            {{ __("Back") }}
          </v-btn>
        </v-card-actions>
      </v-card>
    </v-dialog>
    <v-card
      style="max-height: 70vh; height: 70vh"
      class="cards my-0 py-0 mt-3 grey lighten-5"
    >
      <v-row align="center" class="items px-2 py-1">
        <v-col
          cols="1"
          align="right"
        >
          <v-btn
            icon
            text
            color="error"
            @click="remove_items"
          >
            <v-icon>mdi-delete</v-icon>
            item
          </v-btn>
        </v-col>
        <v-col
          v-if="pos_profile.posa_allow_sales_order && pos_profile.posa_enable_fs_payments"
          cols="7"
          class="pb-2 pr-0"
        >
          <Customer></Customer>
        </v-col>
        <v-col
          v-if="pos_profile.posa_allow_sales_order && !pos_profile.posa_enable_fs_payments"
          cols="9"
          class="pb-2 pr-0"
        >
          <Customer></Customer>
        </v-col>
        <v-col
          v-if="!pos_profile.posa_allow_sales_order && pos_profile.posa_enable_fs_payments"
          cols="9"
          class="pb-2"
        >
          <Customer></Customer>
        </v-col>

        <v-col
          v-if="!pos_profile.posa_allow_sales_order && !pos_profile.posa_enable_fs_payments"
          cols="11"
          class="pb-2"
        >
          <Customer></Customer>
        </v-col>
        <v-col
          v-if="pos_profile.posa_enable_fs_payments"
          cols="1"
          align="left"
        >
          <v-btn
            text icon :color="dynamic_fs_balance_color"
            @click="fs_offline_switch"
          >
            FS<v-icon>{{ dynamic_fs_balance_icon }}</v-icon>
          </v-btn>
        </v-col>
        <v-col
          v-if="pos_profile.posa_enable_fs_payments"
          cols="1"
          align="left"
        >
          <v-btn
            text icon :color="dynamic_pending_icon_color"
            @click="open_pending_fs_bills"
          >
          {{ pending_fs_bills }}<v-icon>mdi-account-clock-outline</v-icon>
          </v-btn>
        </v-col>
        <v-col v-if="pos_profile.posa_allow_sales_order && pos_profile.posa_enable_fs_payments" cols="2" class="pb-2">
          <v-select
            dense
            hide-details
            outlined
            color="primary"
            background-color="white"
            :items="invoiceTypes"
            :label="frappe._('Type')"
            v-model="invoiceType"
            :disabled="invoiceType == 'Return'"
          ></v-select>
        </v-col>
      </v-row>

      <v-row
        align="center"
        class="items px-2 py-1 mt-0 pt-0"
        v-if="pos_profile.posa_use_delivery_charges"
      >
        <v-col cols="8" class="pb-0 mb-0 pr-0 pt-0">
          <v-autocomplete
            dense
            clearable
            auto-select-first
            outlined
            color="primary"
            :label="frappe._('Delivery Charges')"
            v-model="selcted_delivery_charges"
            :items="delivery_charges"
            item-text="name"
            return-object
            background-color="white"
            :no-data-text="__('Charges not found')"
            hide-details
            :filter="deliveryChargesFilter"
            :disabled="readonly"
            @change="update_delivery_charges()"
          >
            <template v-slot:item="data">
              <template>
                <v-list-item-content>
                  <v-list-item-title
                    class="primary--text subtitle-1"
                    v-html="data.item.name"
                  ></v-list-item-title>
                  <v-list-item-subtitle
                    v-html="`Rate: ${data.item.rate}`"
                  ></v-list-item-subtitle>
                </v-list-item-content>
              </template>
            </template>
          </v-autocomplete>
        </v-col>
        <v-col cols="4" class="pb-0 mb-0 pt-0">
          <v-text-field
            dense
            outlined
            color="primary"
            :label="frappe._('Delivery Charges Rate')"
            background-color="white"
            hide-details
            :value="formtCurrency(delivery_charges_rate)"
            :prefix="currencySymbol(pos_profile.currency)"
            disabled
          ></v-text-field>
        </v-col>
      </v-row>
      <v-row
        align="center"
        class="items px-2 py-1 mt-0 pt-0"
        v-if="pos_profile.posa_allow_change_posting_date"
      >
        <v-col
          v-if="pos_profile.posa_allow_change_posting_date"
          cols="4"
          class="pb-2"
        >
          <v-menu
            ref="invoice_posting_date"
            v-model="invoice_posting_date"
            :close-on-content-click="false"
            transition="scale-transition"
            dense
          >
            <template v-slot:activator="{ on, attrs }">
              <v-text-field
                v-model="posting_date"
                :label="frappe._('Posting Date')"
                readonly
                outlined
                dense
                background-color="white"
                clearable
                color="primary"
                hide-details
                v-bind="attrs"
                v-on="on"
              ></v-text-field>
            </template>
            <v-date-picker
              v-model="posting_date"
              no-title
              scrollable
              color="primary"
              :min="
                frappe.datetime.add_days(frappe.datetime.now_date(true), -7)
              "
              :max="frappe.datetime.add_days(frappe.datetime.now_date(true), 7)"
              @input="invoice_posting_date = false"
            >
            </v-date-picker>
          </v-menu>
        </v-col>
      </v-row>

      <div class="my-0 py-0 overflow-y-auto" style="max-height: 60vh">
        <template @mouseover="style = 'cursor: pointer'">
          <v-data-table
            v-model="selected"
            :headers="items_headers"
            :items="items"
            :single-expand="singleExpand"
            :expanded.sync="expanded"
            show-expand
            show-select
            item-key="posa_row_id"
            class="elevation-1"
            :items-per-page="itemsPerPage"
            hide-default-footer
          >
            <template v-slot:item.qty="{ item }">{{
              formtFloat(item.qty)
            }}</template>
            <template v-slot:item.rate="{ item }"
              >{{ currencySymbol(pos_profile.currency) }}
              {{ formtCurrency(item.rate) }}</template
            >
            <template v-slot:item.amount="{ item }"
              >{{ currencySymbol(pos_profile.currency) }}
              {{
                formtCurrency(
                  flt(item.qty, float_precision, undefined, rounding_method) *
                    flt(item.rate, currency_precision, undefined, rounding_method)
                )
              }}</template
            >
            <template v-slot:item.posa_is_offer="{ item }">
              <v-simple-checkbox
                :value="!!item.posa_is_offer || !!item.posa_is_replace"
                disabled
              ></v-simple-checkbox>
            </template>

            <template v-slot:expanded-item="{ headers, item }">
              <td :colspan="headers.length" class="ma-0 pa-0">
                <v-row class="ma-0 pa-0">
                  <v-col cols="1">
                    <v-btn
                      :disabled="!!item.posa_is_offer || !!item.posa_is_replace"
                      icon
                      color="error"
                      @click.stop="remove_item(item)"
                    >
                      <v-icon>mdi-delete</v-icon>
                    </v-btn>
                  </v-col>
                  <v-spacer></v-spacer>
                  <v-col cols="1">
                    <v-btn
                      :disabled="!!item.posa_is_offer || !!item.posa_is_replace"
                      icon
                      color="secondary"
                      @click.stop="subtract_one(item)"
                    >
                      <v-icon>mdi-minus-circle-outline</v-icon>
                    </v-btn>
                  </v-col>
                  <v-col cols="1">
                    <v-btn
                      :disabled="!!item.posa_is_offer || !!item.posa_is_replace"
                      icon
                      color="secondary"
                      @click.stop="add_one(item)"
                    >
                      <v-icon>mdi-plus-circle-outline</v-icon>
                    </v-btn>
                  </v-col>
                </v-row>
                <v-row class="ma-0 pa-0">
                  <v-col cols="4">
                    <v-text-field
                      dense
                      outlined
                      color="primary"
                      :label="frappe._('Item Code')"
                      background-color="white"
                      hide-details
                      v-model="item.item_code"
                      disabled
                    ></v-text-field>
                  </v-col>
                  <v-col cols="4">
                    <v-text-field
                      dense
                      outlined
                      color="primary"
                      :label="frappe._('QTY')"
                      background-color="white"
                      hide-details
                      :value="formtFloat(item.qty)"
                      @change="
                        [
                          setFormatedFloat(item, 'qty', null, false, $event),
                          QTY_textbox_update(item, $event),
                        ]
                      "
                      :rules="[isNumber]"
                      :disabled="!!item.posa_is_offer || !!item.posa_is_replace"
                    ></v-text-field>
                  </v-col>
                  <v-col cols="4">
                    <v-select
                      dense
                      background-color="white"
                      :label="frappe._('UOM')"
                      v-model="item.uom"
                      :items="item.item_uoms"
                      outlined
                      item-text="uom"
                      item-value="uom"
                      hide-details
                      @change="calc_uom(item, $event)"
                      :disabled="
                        !!invoice_doc.is_return ||
                        !!item.posa_is_offer ||
                        !!item.posa_is_replace
                      "
                    >
                    </v-select>
                  </v-col>
                  <v-col cols="4">
                    <v-text-field
                      dense
                      outlined
                      color="primary"
                      :label="frappe._('Rate')"
                      background-color="white"
                      hide-details
                      :prefix="currencySymbol(pos_profile.currency)"
                      :value="formtCurrency(item.rate)"
                      @change="
                        [
                          setFormatedCurrency(
                            item,
                            'rate',
                            null,
                            false,
                            $event
                          ),
                          calc_prices(item, $event),
                        ]
                      "
                      :rules="[isNumber]"
                      id="rate"
                      :disabled="
                        !!item.posa_is_offer ||
                        !!item.posa_is_replace ||
                        !!item.posa_offer_applied ||
                        !pos_profile.posa_allow_user_to_edit_rate ||
                        !!invoice_doc.is_return
                          ? true
                          : false
                      "
                    ></v-text-field>
                  </v-col>
                  <v-col cols="4">
                    <v-text-field
                      dense
                      outlined
                      color="primary"
                      :label="frappe._('Discount Percentage')"
                      background-color="white"
                      hide-details
                      :value="formtFloat(item.discount_percentage)"
                      @change="
                        [
                          setFormatedCurrency(
                            item,
                            'discount_percentage',
                            null,
                            true,
                            $event
                          ),
                          calc_prices(item, $event),
                        ]
                      "
                      :rules="[isNumber]"
                      id="discount_percentage"
                      :disabled="
                        !!item.posa_is_offer ||
                        !!item.posa_is_replace ||
                        item.posa_offer_applied ||
                        !pos_profile.posa_allow_user_to_edit_item_discount ||
                        !!invoice_doc.is_return
                          ? true
                          : false
                      "
                      suffix="%"
                    ></v-text-field>
                  </v-col>
                  <v-col cols="4">
                    <v-text-field
                      dense
                      outlined
                      color="primary"
                      :label="frappe._('Discount Amount')"
                      background-color="white"
                      hide-details
                      :value="formtCurrency(item.discount_amount)"
                      :rules="[isNumber]"
                      @change="
                        [
                          setFormatedCurrency(
                            item,
                            'discount_amount',
                            null,
                            true,
                            $event
                          ),
                          ,
                          calc_prices(item, $event),
                        ]
                      "
                      :prefix="currencySymbol(pos_profile.currency)"
                      id="discount_amount"
                      :disabled="
                        !!item.posa_is_offer ||
                        !!item.posa_is_replace ||
                        !!item.posa_offer_applied ||
                        !pos_profile.posa_allow_user_to_edit_item_discount ||
                        !!invoice_doc.is_return
                          ? true
                          : false
                      "
                    ></v-text-field>
                  </v-col>
                  <v-col cols="4">
                    <v-text-field
                      dense
                      outlined
                      color="primary"
                      :label="frappe._('Price list Rate')"
                      background-color="white"
                      hide-details
                      :value="formtCurrency(item.price_list_rate)"
                      disabled
                      :prefix="currencySymbol(pos_profile.currency)"
                    ></v-text-field>
                  </v-col>
                  <v-col cols="4">
                    <v-text-field
                      dense
                      outlined
                      color="primary"
                      :label="frappe._('Available QTY')"
                      background-color="white"
                      hide-details
                      :value="formtFloat(item.actual_qty)"
                      disabled
                    ></v-text-field>
                  </v-col>
                  <v-col cols="4">
                    <v-text-field
                      dense
                      outlined
                      color="primary"
                      :label="frappe._('Group')"
                      background-color="white"
                      hide-details
                      v-model="item.item_group"
                      disabled
                    ></v-text-field>
                  </v-col>
                  <v-col cols="4">
                    <v-text-field
                      dense
                      outlined
                      color="primary"
                      :label="frappe._('Stock QTY')"
                      background-color="white"
                      hide-details
                      :value="formtFloat(item.stock_qty)"
                      disabled
                    ></v-text-field>
                  </v-col>
                  <v-col cols="4">
                    <v-text-field
                      dense
                      outlined
                      color="primary"
                      :label="frappe._('Stock UOM')"
                      background-color="white"
                      hide-details
                      v-model="item.stock_uom"
                      disabled
                    ></v-text-field>
                  </v-col>
                  <v-col align="center" cols="4" v-if="item.posa_offer_applied">
                    <v-checkbox
                      dense
                      :label="frappe._('Offer Applied')"
                      v-model="item.posa_offer_applied"
                      readonly
                      hide-details
                      class="shrink mr-2 mt-0"
                    ></v-checkbox>
                  </v-col>
                  <v-col
                    cols="4"
                    v-if="item.has_serial_no == 1 || item.serial_no"
                  >
                    <v-text-field
                      dense
                      outlined
                      color="primary"
                      :label="frappe._('Serial No QTY')"
                      background-color="white"
                      hide-details
                      v-model="item.serial_no_selected_count"
                      type="number"
                      disabled
                    ></v-text-field>
                  </v-col>
                  <v-col
                    cols="12"
                    v-if="item.has_serial_no == 1 || item.serial_no"
                  >
                    <v-autocomplete
                      v-model="item.serial_no_selected"
                      :items="item.serial_no_data"
                      item-text="serial_no"
                      outlined
                      dense
                      chips
                      color="primary"
                      small-chips
                      :label="frappe._('Serial No')"
                      multiple
                      @change="set_serial_no(item)"
                    ></v-autocomplete>
                  </v-col>
                  <v-col
                    cols="4"
                    v-if="item.has_batch_no == 1 || item.batch_no"
                  >
                    <v-text-field
                      dense
                      outlined
                      color="primary"
                      :label="frappe._('Batch No. Available QTY')"
                      background-color="white"
                      hide-details
                      :value="formtFloat(item.actual_batch_qty)"
                      disabled
                    ></v-text-field>
                  </v-col>
                  <v-col
                    cols="4"
                    v-if="item.has_batch_no == 1 || item.batch_no"
                  >
                    <v-text-field
                      dense
                      outlined
                      color="primary"
                      :label="frappe._('Batch No Expiry Date')"
                      background-color="white"
                      hide-details
                      v-model="item.batch_no_expiry_date"
                      disabled
                    ></v-text-field>
                  </v-col>
                  <v-col
                    cols="8"
                    v-if="item.has_batch_no == 1 || item.batch_no"
                  >
                    <v-autocomplete
                      v-model="item.batch_no"
                      :items="item.batch_no_data"
                      item-text="batch_no"
                      outlined
                      dense
                      color="primary"
                      :label="frappe._('Batch No')"
                      @change="set_batch_qty(item, $event)"
                    >
                      <template v-slot:item="data">
                        <template>
                          <v-list-item-content>
                            <v-list-item-title
                              v-html="data.item.batch_no"
                            ></v-list-item-title>
                            <v-list-item-subtitle
                              v-html="
                                `Available QTY  '${data.item.batch_qty}' - Expiry Date ${data.item.expiry_date}`
                              "
                            ></v-list-item-subtitle>
                          </v-list-item-content>
                        </template>
                      </template>
                    </v-autocomplete>
                  </v-col>

                  <v-col
                    cols="3"
                    v-show="display_pending_bill_details"
                  >
                    <v-text-field
                      dense
                      outlined
                      color="primary"
                      :label="frappe._('Invoice Name')"
                      background-color="white"
                      hide-details
                      v-model="invoice_doc.name"
                      disabled
                    ></v-text-field>
                  </v-col>
                  <v-col
                    cols="3"
                    v-show="display_pending_bill_details"
                  >
                    <v-text-field
                      dense
                      outlined
                      color="primary"
                      :label="frappe._('Invoice Status')"
                      background-color="white"
                      hide-details
                      v-model="invoice_doc.status"
                      disabled
                    ></v-text-field>
                  </v-col>
                  <v-col
                    cols="3"
                    v-show="display_pending_bill_details"
                  >
                    <v-text-field
                      dense
                      outlined
                      color="primary"
                      :label="frappe._('Doc Status')"
                      background-color="white"
                      hide-details
                      v-model="invoice_doc.docstatus"
                      disabled
                    ></v-text-field>
                  </v-col>
                  <v-col
                    cols="3"
                    v-show="display_pending_bill_details"
                  >
                    <v-text-field
                      dense
                      outlined
                      color="primary"
                      :label="frappe._('FS Transfer Status')"
                      background-color="white"
                      hide-details
                      v-model="invoice_doc.custom_fs_transfer_status"
                      disabled
                    ></v-text-field>
                  </v-col>

                  <v-col
                    cols="4"
                    v-if="
                      pos_profile.posa_allow_sales_order &&
                      invoiceType == 'Order'
                    "
                  >
                    <v-menu
                      ref="item_delivery_date"
                      v-model="item.item_delivery_date"
                      :close-on-content-click="false"
                      :return-value.sync="item.posa_delivery_date"
                      transition="scale-transition"
                      dense
                    >
                      <template v-slot:activator="{ on, attrs }">
                        <v-text-field
                          v-model="item.posa_delivery_date"
                          :label="frappe._('Delivery Date')"
                          readonly
                          outlined
                          dense
                          clearable
                          color="primary"
                          hide-details
                          v-bind="attrs"
                          v-on="on"
                        ></v-text-field>
                      </template>
                      <v-date-picker
                        v-model="item.posa_delivery_date"
                        no-title
                        scrollable
                        color="primary"
                        :min="frappe.datetime.now_date()"
                      >
                        <v-spacer></v-spacer>
                        <v-btn
                          text
                          color="primary"
                          @click="item.item_delivery_date = false"
                        >
                          Cancel
                        </v-btn>
                        <v-btn
                          text
                          color="primary"
                          @click="
                            [
                              $refs.item_delivery_date.save(
                                item.posa_delivery_date
                              ),
                              validate_due_date(item),
                            ]
                          "
                        >
                          OK
                        </v-btn>
                      </v-date-picker>
                    </v-menu>
                  </v-col>
                  <v-col
                    cols="8"
                    v-if="pos_profile.posa_display_additional_notes"
                  >
                    <v-textarea
                      class="pa-0"
                      outlined
                      dense
                      clearable
                      color="primary"
                      auto-grow
                      rows="1"
                      :label="frappe._('Additional Notes')"
                      v-model="item.posa_notes"
                      :value="item.posa_notes"
                    ></v-textarea>
                  </v-col>
                </v-row>
              </td>
            </template>
          </v-data-table>
        </template>
      </div>
    </v-card>
    <v-card v-if="pos_profile.posa_enable_fs_payments"
      class="cards mb-0 mt-3 py-0 grey lighten-5">
      <v-row no-gutters>
        <v-col cols="5">
          <v-row no-gutters class="pa-1 pt-9 pr-1">
            <v-col cols="6" class="pa-1">
              <v-text-field
                :value="formtFloat(total_qty)"
                :label="frappe._('Total Qty')"
                outlined
                dense
                readonly
                hide-details
                color="accent"
              ></v-text-field>
            </v-col>
            <v-col
              v-if="!pos_profile.posa_use_percentage_discount"
              cols="6"
              class="pa-1"
            >
              <v-text-field
                :value="formtCurrency(discount_amount)"
                @change="
                  setFormatedCurrency(
                    discount_amount,
                    'discount_amount',
                    null,
                    false,
                    $event
                  )
                "
                :rules="[isNumber]"
                :label="frappe._('Additional Discount')"
                ref="discount"
                outlined
                dense
                hide-details
                color="warning"
                :prefix="currencySymbol(pos_profile.currency)"
                :disabled="
                  !pos_profile.posa_allow_user_to_edit_additional_discount ||
                  discount_percentage_offer_name
                    ? true
                    : false
                "
              ></v-text-field>
            </v-col>
            <v-col
              v-if="pos_profile.posa_use_percentage_discount"
              cols="6"
              class="pa-1"
            >
              <v-text-field
                :value="formtFloat(additional_discount_percentage)"
                @change="
                  [
                    setFormatedFloat(
                      additional_discount_percentage,
                      'additional_discount_percentage',
                      null,
                      false,
                      $event
                    ),
                    update_discount_umount(),
                  ]
                "
                :rules="[isNumber]"
                :label="frappe._('Additional Discount %')"
                suffix="%"
                ref="percentage_discount"
                outlined
                dense
                color="warning"
                hide-details
                :disabled="
                  !pos_profile.posa_allow_user_to_edit_additional_discount ||
                  discount_percentage_offer_name
                    ? true
                    : false
                "
              ></v-text-field>
            </v-col>
            <v-col cols="6" class="pa-1 mt-2">
              <v-text-field
                :value="formtCurrency(total_items_discount_amount)"
                :prefix="currencySymbol(pos_profile.currency)"
                :label="frappe._('Items Discounts')"
                outlined
                dense
                color="warning"
                readonly
                hide-details
              ></v-text-field>
            </v-col>

            <v-col cols="6" class="pa-1 mt-2">
              <v-text-field
                :value="formtCurrency(subtotal)"
                :prefix="currencySymbol(pos_profile.currency)"
                :label="frappe._('Total')"
                outlined
                dense
                readonly
                hide-details
                color="success"
              ></v-text-field>
              <!--v-text-field
                :value="formtCurrency_amount(subtotal)"
                :prefix="currencySymbol(pos_profile.currency)"
                :label="frappe._('Total')"
                outlined
                dense
                readonly
                hide-details
                color="success"
              ></v-text-field-->
            </v-col>
          </v-row>
        </v-col>
        <v-col cols="7">
          <v-row no-gutters class="pa-1 pt-2 pl-0">
            <v-col v-if="pos_profile.custom_allow_select_sales_order === 1"
              cols="4" class="pa-1">
              <v-btn
                block
                class="pa-0"
                color="warning"
                dark
                @click="get_draft_invoices"
                >{{ __("Held") }}</v-btn
              >
            </v-col>
            <v-col v-if="pos_profile.custom_allow_select_sales_order !== 1"
              cols="6" class="pa-1">
              <v-btn
                block
                class="pa-0"
                color="warning"
                dark
                @click="get_draft_invoices"
                >{{ __("Held") }}</v-btn
              >
            </v-col>
            <v-col
              v-if="pos_profile.custom_allow_select_sales_order === 1"
              cols="4"
              class="pa-1"
            >
              <v-btn
                block
                class="pa-0"
                color="info"
                dark
                @click="get_draft_orders"
                >{{ __("Select S.O") }}</v-btn
              >
            </v-col>
            <v-col
              v-if="pos_profile.posa_allow_print_draft_invoices && pos_profile.custom_allow_select_sales_order === 1"
              cols="4"
              class="pa-1"
            >
              <v-btn
                block
                class="pa-0"
                color="primary"
                @click="print_draft_invoice"
                dark
                >{{ __("Print Draft") }}</v-btn
              >
            </v-col>
            <v-col
              v-if="pos_profile.posa_allow_print_draft_invoices && pos_profile.custom_allow_select_sales_order !== 1"
              cols="6"
              class="pa-1"
            >
              <v-btn
                block
                class="pa-0"
                color="primary"
                @click="print_draft_invoice"
                dark
                >{{ __("Print Draft") }}</v-btn
              >
            </v-col>
            <v-col cols="6" class="pa-1">
              <v-btn
                block
                class="pa-0"
                color="error"
                dark
                @click="cancel_dialog = true"
                >{{ __("Cancel") }}</v-btn
              >
            </v-col>
            <v-col cols="6" class="pa-1">
              <v-btn
                block
                class="pa-0"
                :class="{ 'disable-events': !pos_profile.posa_allow_return }"
                color="secondary"
                dark
                @click="open_returns"
                >{{ __("Return") }}</v-btn
              >
            </v-col>
            <v-col class="pa-1">
              <v-btn
                block
                class="pa-0"
                color="success"
                @click="show_payment"
                ref="checkout"
                dark
                >{{ __("PAY / Create S.O") }}</v-btn
              >
            </v-col>
            <v-col cols="6" class="pa-1">
              <v-btn
                block
                class="pa-0"
                color="accent"
                dark
                @click="new_invoice"
                >{{ __("Hold Bill") }}</v-btn
              >
            </v-col>
            <!-- <v-col
              cols="6"
              class="pa-1"
            >
              <v-btn
                block
                class="pa-0"
                color="primary"
                @click="offline_fs_pay"
                dark
                >{{ __("Offline FS PAY") }}</v-btn
              >
            </v-col> -->
          </v-row>
        </v-col>
      </v-row>
    </v-card>
    <v-card v-if="!pos_profile.posa_enable_fs_payments"
      class="cards mb-0 mt-3 py-0 grey lighten-5">
      <v-row no-gutters>
        <v-col cols="6">
          <v-row no-gutters class="pa-1 pt-9 pr-1">
            <v-col cols="6" class="pa-1">
              <v-text-field
                :value="formtFloat(total_qty)"
                :label="frappe._('Total Qty')"
                outlined
                dense
                readonly
                hide-details
                color="accent"
              ></v-text-field>
            </v-col>
            <v-col
              v-if="!pos_profile.posa_use_percentage_discount"
              cols="6"
              class="pa-1"
            >
              <v-text-field
                :value="formtCurrency(discount_amount)"
                @change="
                  setFormatedCurrency(
                    discount_amount,
                    'discount_amount',
                    null,
                    false,
                    $event
                  )
                "
                :rules="[isNumber]"
                :label="frappe._('Additional Discount')"
                ref="discount"
                outlined
                dense
                hide-details
                color="warning"
                :prefix="currencySymbol(pos_profile.currency)"
                :disabled="
                  !pos_profile.posa_allow_user_to_edit_additional_discount ||
                  discount_percentage_offer_name
                    ? true
                    : false
                "
              ></v-text-field>
            </v-col>
            <v-col
              v-if="pos_profile.posa_use_percentage_discount"
              cols="6"
              class="pa-1"
            >
              <v-text-field
                :value="formtFloat(additional_discount_percentage)"
                @change="
                  [
                    setFormatedFloat(
                      additional_discount_percentage,
                      'additional_discount_percentage',
                      null,
                      false,
                      $event
                    ),
                    update_discount_umount(),
                  ]
                "
                :rules="[isNumber]"
                :label="frappe._('Additional Discount %')"
                suffix="%"
                ref="percentage_discount"
                outlined
                dense
                color="warning"
                hide-details
                :disabled="
                  !pos_profile.posa_allow_user_to_edit_additional_discount ||
                  discount_percentage_offer_name
                    ? true
                    : false
                "
              ></v-text-field>
            </v-col>
            <v-col cols="6" class="pa-1 mt-2">
              <v-text-field
                :value="formtCurrency(total_items_discount_amount)"
                :prefix="currencySymbol(pos_profile.currency)"
                :label="frappe._('Items Discounts')"
                outlined
                dense
                color="warning"
                readonly
                hide-details
              ></v-text-field>
            </v-col>

            <v-col cols="6" class="pa-1 mt-2">
              <v-text-field
                :value="formtCurrency(subtotal)"
                :prefix="currencySymbol(pos_profile.currency)"
                :label="frappe._('Total')"
                outlined
                dense
                readonly
                hide-details
                color="success"
              ></v-text-field>
            </v-col>
          </v-row>
        </v-col>
        <v-col cols="6">
          <v-row no-gutters class="pa-1 pt-2 pl-0">
            <v-col cols="6" class="pa-1">
              <v-btn
                block
                class="pa-0"
                color="warning"
                dark
                @click="get_draft_invoices"
                >{{ __("Held") }}</v-btn
              >
            </v-col>
            <v-col
              v-if="pos_profile.custom_allow_select_sales_order === 1"
              cols="6"
              class="pa-1"
            >
              <v-btn
                block
                class="pa-0"
                color="info"
                dark
                @click="get_draft_orders"
                >{{ __("Select S.O") }}</v-btn
              >
            </v-col>
            <v-col cols="6" class="pa-1">
              <v-btn
                block
                class="pa-0"
                :class="{ 'disable-events': !pos_profile.posa_allow_return }"
                color="secondary"
                dark
                @click="open_returns"
                >{{ __("Return") }}</v-btn
              >
            </v-col>
            <v-col cols="6" class="pa-1">
              <v-btn
                block
                class="pa-0"
                color="error"
                dark
                @click="cancel_dialog = true"
                >{{ __("Cancel") }}</v-btn
              >
            </v-col>
            <v-col cols="6" class="pa-1">
              <v-btn
                block
                class="pa-0"
                color="accent"
                dark
                @click="new_invoice"
                >{{ __("Hold") }}</v-btn
              >
            </v-col>
            <v-col class="pa-1">
              <v-btn
                block
                class="pa-0"
                color="success"
                @click="show_payment"
                ref="checkout"
                dark
                >{{ __("PAY / Create S.O") }}</v-btn
              >
            </v-col>
            <!-- <v-col
              v-if="pos_profile.posa_allow_print_draft_invoices"
              cols="6"
              class="pa-1"
            >
              <v-btn
                block
                class="pa-0"
                color="primary"
                @click="print_draft_invoice"
                dark
                >{{ __("Print Draft") }}</v-btn
              >
            </v-col> -->
            <v-col
              cols="6"
              class="pa-1"
            >
              <v-btn
                block
                class="pa-0"
                color="primary"
                @click="save_as_order"
                dark
                >{{ __("Save as Order") }}</v-btn
              >
            </v-col>
          </v-row>
        </v-col>
      </v-row>
    </v-card>
  </div>
</template>

<script>
import { evntBus } from "../../bus";
import format from "../../format";
import Customer from "./Customer.vue";

export default {
  mixins: [format],
  data() {
    return {
      pos_profile: "",
      pos_opening_shift: "",
      stock_settings: "",
      invoice_doc: "",
      return_doc: "",
      customer: "",
      customer_info: "",
      discount_amount: 0,
      additional_discount_percentage: 0,
      total_tax: 0,
      items: [],
      posOffers: [],
      posa_offers: [],
      posa_coupons: [],
      allItems: [],
      discount_percentage_offer_name: null,
      invoiceTypes: ["Invoice", "Order"],
      invoiceType: "Invoice",
      itemsPerPage: 1000,
      expanded: [],
      singleExpand: true,
      cancel_dialog: false,
      float_precision: 2,
      currency_precision: 2,
      rounding_method: "", // for pulling the 'rounding method' value from ERPNext
      balance_available: null, // Customer FS Account balance
      dynamic_fs_balance_color: 'grey',  // grey,error,success (availability of FS balance)
      dynamic_fs_balance_icon: 'mdi-bank-off', // 'mdi-bank' (availability of FS balance)
      dynamic_pending_icon_color: 'grey', // highlights pending offline bills
      pending_fs_bills: 0,
      fs_transfer_pending: false, // for 'Offline FS Pay'
      fs_offline: false, // to manually switch off FS Balance checks
      display_pending_bill_details: false, // toggles display of pending bill details
      new_line: false,
      delivery_charges: [],
      delivery_charges_rate: 0,
      selcted_delivery_charges: {},
      invoice_posting_date: false,
      posting_date: frappe.datetime.nowdate(),
      items_headers: [
        {
          text: __("Name"),
          align: "start",
          sortable: true,
          value: "item_name",
        },
        { text: __('Code'), value: 'item_code', align: 'center' },
        { text: __("QTY"), value: "qty", align: "center" },
        { text: __("UOM"), value: "uom", align: "center" },
        { text: __("Rate"), value: "rate", align: "center" },
        { text: __("Amount"), value: "amount", align: "center" },
        // { text: __("is Offer"), value: "posa_is_offer", align: "center" },
      ],
      selected: [], // for v-data-table row selections
    };
  },

  components: {
    Customer,
  },

  computed: {
    total_qty() {
      this.close_payments();
      let qty = 0;
      this.items.forEach((item) => {
        qty += flt(item.qty);
      });
      return this.flt(qty, this.float_precision);
    },
    Total() {
      let sum = 0;
      this.items.forEach((item) => {
        sum += flt(item.qty) * flt(item.rate);
      });
      return this.flt(sum, this.currency_precision);
    },
    subtotal() {
      this.close_payments();
      let sum = 0;
      this.items.forEach((item) => {
        sum += flt(item.qty) * flt(item.rate);
      });
      sum -= this.flt(this.discount_amount);
      sum += this.flt(this.delivery_charges_rate);
      return this.flt(sum, this.currency_precision, undefined, this.rounding_method);
      //let return_sum = this.bankers_rounding(sum);
      //return return_sum;
    },
    total_items_discount_amount() {
      let sum = 0;
      this.items.forEach((item) => {
        sum += flt(item.qty) * flt(item.discount_amount);
      });
      return this.flt(sum, this.float_precision);
    },
  },

  methods: {
    fs_offline_switch() {
      this.fs_offline = !this.fs_offline;
      if (this.fs_offline) {
        evntBus.$emit('show_mesage', {
          text: 'FS Offline',
          color: 'warning',
        });
        this.reset_fs_variables(this.fs_offline);
      }
      else {
        evntBus.$emit('show_mesage', {
          text: 'FS Online',
          color: 'success',
        });
        this.reset_fs_variables(this.fs_offline);
      }
      evntBus.$emit('fs_offline', this.fs_offline);
    },
    reset_fs_variables(fs_offline = false) {
      this.balance_available = null;
      console.log("Balance: ", this.balance_available);
      if (fs_offline) {
        console.log("fs_offline: ", fs_offline);
        this.dynamic_fs_balance_color = 'warning';
        this.dynamic_fs_balance_icon = 'mdi-bank-off';
      }
      else {
        console.log("fs_offline: ", fs_offline);
        this.reset_fs_balance_status();
      }
      this.reset_pending_fs_bills_status();
    },
    fs_balance_check(fs_acc_customer) {
      const vm = this;
      frappe.call({
        method: 'payments.payment_gateways.doctype.fs_settings.fs_settings.get_account_max_amount',
        args: {fs_acc_customer: fs_acc_customer},
        callback: function (r) {
          if (r.message) {
            if (r.message['Result'] == 'OK') {
              vm.balance_available = parseFloat(r.message['maxAmount']);
              console.log('vm.balance_available: ', vm.balance_available);
              if (vm.balance_available > 0) {
                vm.dynamic_fs_balance_color = 'success';
                vm.dynamic_fs_balance_icon = 'mdi-bank';
                evntBus.$emit('balance_available', vm.balance_available); // sent to Payments.Vue
              }
              else if (vm.balance_available === 0) {
                vm.dynamic_fs_balance_color = 'error';
                vm.dynamic_fs_balance_icon = 'mdi-bank';
                evntBus.$emit('show_mesage', {
                  text: 'Insufficient Balance',
                  color: 'warning',
                });
                evntBus.$emit('balance_available', vm.balance_available);
              }
              else {
                evntBus.$emit('show_mesage', {
                  text: 'Please verify the FS Account Number for this Customer',
                  color: 'error',
                });
                vm.new_invoice(); // resets the flow
              }
            }
            else {
              evntBus.$emit('show_mesage', {
                text: r.message['Result'],
                color: 'error',
              });
              vm.dynamic_fs_balance_color = 'error';
              vm.dynamic_fs_balance_icon = 'mdi-bank';
              vm.new_invoice(); // resets the flow
            }
          }
          else {
            vm.dynamic_fs_balance_color = 'grey';
            vm.dynamic_fs_balance_icon = 'mdi-bank-off';
          }
        }
      })
    },
    reset_fs_balance_status() {
      this.dynamic_fs_balance_color = 'grey';
      this.dynamic_fs_balance_icon = 'mdi-bank-off';
    },
    pending_fs_bills_check(customer) {
      const vm = this;
      frappe.call({
        method: 'posawesome.posawesome.api.posapp.pending_fs_bills_query',
        args: {
          customer: customer,
          company: vm.pos_profile.company
        },
        callback: (r) => {
          if (r.message) {
              if (r.message > 0) {
              this.dynamic_pending_icon_color = 'warning';
              this.pending_fs_bills = r.message;
            }
          }
          else {
            vm.dynamic_pending_icon_color = 'grey';
          }
        }
      })
    },
    reset_pending_fs_bills_status() {
      this.dynamic_pending_icon_color = 'grey';
      this.pending_fs_bills = 0;
    },
    get_customer_type(customer) {
      const vm = this;
      frappe.call({
        method: 'posawesome.posawesome.api.posapp.get_customer_type',
        args: {
          customer: customer
        },
        callback: (r) => {
          if (r.message) {
            if (r.message == "Company")
              vm.invoiceType = "Order";
            else vm.invoiceType = "Invoice";
          }
        }
      })
    },
    remove_item(item) {
      const index = this.items.findIndex(
        (el) => el.posa_row_id == item.posa_row_id
      );
      if (index >= 0) {
        this.items.splice(index, 1);
      }
      const idx = this.expanded.findIndex(
        (el) => el.posa_row_id == item.posa_row_id
      );
      if (idx >= 0) {
        this.expanded.splice(idx, 1);
      }
    },

    remove_items() {
      for (let i = 0; i < this.selected.length; i++) {
        const index = this.items.indexOf(this.selected[i]);
        this.items.splice(index, 1);
      }
      this.selected = [];  // reset the old item selections after deletion
    },

    add_one(item) {
      item.qty++;
      if (item.qty == 0) {
        this.remove_item(item);
      }
      this.calc_stock_qty(item, item.qty);
      // in case custom_item_add_on is set, then add the add-on item
      if (item.custom_item_add_on) {
        let index = this.items.findIndex(
          (el) =>
            el.item_code === item.custom_item_add_on
        );
        let add_on_item = this.items[index];
        add_on_item.qty++;
        if (add_on_item.qty == 0) {
        this.remove_item(add_on_item);
        }
        this.calc_stock_qty(add_on_item, add_on_item.qty);
      }
      this.$forceUpdate();
    },
    subtract_one(item) {
      item.qty--;
      if (item.qty == 0) {
        this.remove_item(item);
      }
      this.calc_stock_qty(item, item.qty);
      // in case custom_item_add_on is set, then subtract the add-on item
      if (item.custom_item_add_on) {
        let index = this.items.findIndex(
          (el) =>
            el.item_code === item.custom_item_add_on
        );
        let add_on_item = this.items[index];
        add_on_item.qty--;
        if (add_on_item.qty == 0) {
        this.remove_item(add_on_item);
        }
        this.calc_stock_qty(add_on_item, add_on_item.qty);
      }
      this.$forceUpdate();
    },

    add_item(item) {
      let item_to_add;
      // adding this variable/object as when we do update_item_detail directly on the item object (in order to set item.batch_no)
      // then the array value items[].actual_batch_qty gets reset to '', and some parts of the code logic below fails..
      if (item.has_batch_no && !item.batch_no) {
        item_to_add = { ...item };
        this.update_item_detail(item_to_add);
      }
      if (!item.uom) {
        item.uom = item.stock_uom;
      }
      let index = -1;
      if (!this.new_line && item.has_batch_no) { // matching items with same item_code and batch, to be put in the same line with increased qty.
        index = this.items.findIndex(
          (el) =>
            el.item_code === item.item_code &&
            el.uom === item.uom &&
            !el.posa_is_offer &&
            !el.posa_is_replace &&
            //el.batch_no === item.batch_no
            el.batch_no === item_to_add.batch_no
        );
        //console.log(item.item_code, item.uom, item_to_add.batch_no);
        //console.log("index: ", index);
      } else {
        index = this.items.findIndex(
          (el) =>
            el.item_code === item.item_code &&
            el.uom === item.uom &&
            !el.posa_is_offer &&
            !el.posa_is_replace &&
            el.batch_no === item.batch_no
            //el.batch_no === item_to_add.batch_no
        );
        //console.log(item.item_code, item.uom, item.batch_no);
        //console.log("index: ", index);
      }
      if (index === -1 || this.new_line) {
        //console.log("Label-A");
        const new_item = this.get_new_item(item);
        if (item.has_serial_no && item.to_set_serial_no) {
          new_item.serial_no_selected = [];
          new_item.serial_no_selected.push(item.to_set_serial_no);
          item.to_set_serial_no = null;
        }
        if (item.has_batch_no && item.to_set_batch_no) {
          //console.log("Label-B");
          new_item.batch_no = item.to_set_batch_no;
          item.to_set_batch_no = null;
          item.batch_no = null;
          this.set_batch_qty(new_item, new_item.batch_no, false);
        }
        this.items.unshift(new_item);
        this.update_item_detail(new_item);
      } else {
        //console.log("Label-C");
        const cur_item = this.items[index];
        this.update_items_details([cur_item]);
        if (item.has_serial_no && item.to_set_serial_no) {
          if (cur_item.serial_no_selected.includes(item.to_set_serial_no)) {
            evntBus.$emit("show_mesage", {
              text: __(`This Serial Number {0} has already been added!`, [
                item.to_set_serial_no,
              ]),
              color: "warning",
            });
            item.to_set_serial_no = null;
            return;
          }
          cur_item.serial_no_selected.push(item.to_set_serial_no);
          item.to_set_serial_no = null;
        }
        if (!cur_item.has_batch_no) {
          //console.log("Label-D");
          //cur_item.qty += item.qty || 1;
          // type-casting all fields to Float because when the item qty is edited from the invoice (using QTY_textbox_update), -
          // the item qty fields are converted to Float, while the cur_item qty remains an integer - due to this the addition was happenning as text concatenation.
          cur_item.qty = parseFloat(cur_item.qty);
          cur_item.qty += parseFloat(item.qty) || parseFloat(1);
          this.calc_stock_qty(cur_item, cur_item.qty);
        } else {
          //console.log("Label-E");
          if (
            (cur_item.stock_qty < cur_item.actual_batch_qty &&
              cur_item.batch_no == item_to_add.batch_no) ||
            !cur_item.batch_no
          ) {
            //console.log("Label-F");
            //cur_item.qty += item.qty || 1;
            // type-casting all fields to Float because when the item qty is edited from the invoice (using QTY_textbox_update), -
            // the item qty fields are converted to Float, while the cur_item qty remains an integer - due to this the addition was happenning as text concatenation.
            cur_item.qty = parseFloat(cur_item.qty);
            cur_item.qty += parseFloat(item.qty) || parseFloat(1);
            this.calc_stock_qty(cur_item, cur_item.qty);
          } else {
            //console.log("Label-G");
            const new_item = this.get_new_item(cur_item);
            new_item.batch_no = item.batch_no || item.to_set_batch_no;
            new_item.batch_no_expiry_date = "";
            new_item.actual_batch_qty = "";
            new_item.qty = item.qty || 1;
            if (new_item.batch_no) {
              //console.log("Label-H");
              this.set_batch_qty(new_item, new_item.batch_no, false);
              item.to_set_batch_no = null;
              item.batch_no = null;
            }
            this.items.unshift(new_item);
          }
        }
        this.set_serial_no(cur_item);
      }
      this.$forceUpdate();
    },

    get_new_item(item) {
      const new_item = { ...item };
      if (!item.qty) {
        item.qty = 1;
      }
      if (!item.posa_is_offer) {
        item.posa_is_offer = 0;
      }
      if (!item.posa_is_replace) {
        item.posa_is_replace = "";
      }
      new_item.stock_qty = item.qty;
      new_item.discount_amount = 0;
      new_item.discount_percentage = 0;
      new_item.discount_amount_per_item = 0;
      new_item.price_list_rate = item.rate;
      new_item.qty = item.qty;
      new_item.uom = item.uom ? item.uom : item.stock_uom;
      new_item.actual_batch_qty = "";
      new_item.conversion_factor = 1;
      new_item.posa_offers = JSON.stringify([]);
      new_item.posa_offer_applied = 0;
      new_item.posa_is_offer = item.posa_is_offer;
      new_item.posa_is_replace = item.posa_is_replace || null;
      new_item.is_free_item = 0;
      new_item.posa_notes = "";
      new_item.posa_delivery_date = "";
      new_item.posa_row_id = this.makeid(20);
      if (
        (!this.pos_profile.posa_auto_set_batch && new_item.has_batch_no) ||
        new_item.has_serial_no
      ) {
        this.expanded.push(new_item);
      }
      return new_item;
    },

    cancel_invoice() {
      const doc = this.get_invoice_doc();
      this.invoiceType = this.pos_profile.posa_default_sales_order
        ? "Order"
        : "Invoice";
      this.invoiceTypes = ["Invoice", "Order"];
      this.posting_date = frappe.datetime.nowdate();
      if (doc.name && this.pos_profile.posa_allow_delete) {
        frappe.call({
          method: "posawesome.posawesome.api.posapp.delete_invoice",
          args: { invoice: doc.name },
          async: true,
          callback: function (r) {
            if (r.message) {
              evntBus.$emit("show_mesage", {
                text: r.message,
                color: "warning",
              });
            }
          },
        });
      }
      this.items = [];
      this.selected = [];  // reset the old item selections when a new invoice loads
      this.posa_offers = [];
      evntBus.$emit("set_pos_coupons", []);
      this.posa_coupons = [];
      this.customer = this.pos_profile.customer;
      this.invoice_doc = "";
      this.return_doc = "";
      this.discount_amount = 0;
      this.additional_discount_percentage = 0;
      this.delivery_charges_rate = 0;
      this.selcted_delivery_charges = {};
      evntBus.$emit("set_customer_readonly", false);
      this.reset_fs_balance_status();
      this.reset_pending_fs_bills_status();
      evntBus.$emit('input_customer'); // pass event to Customer.vue
      this.cancel_dialog = false;
    },

    offline_fs_pay() {
      if (this.fs_offline) {
        this.offline_save();
      }
      else {
        const vm = this;
        frappe.call({
          method: 'payments.payment_gateways.doctype.fs_settings.fs_settings.login',
          callback: function (r) {
            if (r.message !== 'OK')
              vm.offline_save(); // FS is Offline
            else {
              evntBus.$emit('show_mesage', {
                text: 'FS is Online, please press the Pay button instead',
                color: 'error',
              });
            }
          },
        });
      }
    },

    async offline_save() {
      if (!this.customer) {
        evntBus.$emit("show_mesage", {
          text: __(`There is no Customer !`),
          color: "error",
        });
        return;
      }
      if (!this.items.length) {
        evntBus.$emit("show_mesage", {
          text: __(`There are no Items !`),
          color: "error",
        });
        return;
      }
      this.fs_transfer_pending = true;
      let await_new_invoice = await this.new_invoice();
      evntBus.$emit('show_mesage', {
        text: 'Offline FS Invoice saved',
        color: 'info',
      });
    },

    new_invoice(data = {}) {
      let old_invoice = null;
      evntBus.$emit("set_customer_readonly", false);
      this.expanded = [];
      this.selected = [];  // reset the old item selections when a new invoice loads
      this.posa_offers = [];
      evntBus.$emit("set_pos_coupons", []);
      this.posa_coupons = [];
      this.return_doc = "";
      const doc = this.get_invoice_doc();
      doc.items.forEach((item) => {
        if ((!item.rate || item.rate == 0) && !this.pos_profile.posa_allow_zero_rated_items) {
          frappe.throw("Rate cannot be zero for any item");
        }
      })
      if (this.fs_transfer_pending) doc.custom_fs_transfer_status = "PENDING"; // for 'Offline FS Pay'
      if (doc.name) {
        old_invoice = this.update_invoice(doc);
        //console.log("A: old_invoice: ", old_invoice);
      } else {
        if (doc.items.length) {
          old_invoice = this.update_invoice(doc);
          //console.log("B: old_invoice: ", old_invoice);
        }
      }
      if (!data.name && !data.is_return) {
        this.items = [];
        this.customer = this.pos_profile.customer;
        this.invoice_doc = "";
        this.fs_transfer_pending = false; // for 'Offline FS Pay'
        this.discount_amount = 0;
        this.additional_discount_percentage = 0;
        this.invoiceType = this.pos_profile.posa_default_sales_order
          ? "Order"
          : "Invoice";
        this.invoiceTypes = ["Invoice", "Order"];
        if (!this.display_pending_bill_details) {
          // need to retain these variables in case pending bill details are pulled.
          this.reset_fs_balance_status();
          this.reset_pending_fs_bills_status();
        }
        evntBus.$emit('input_customer'); // pass event to Customer.vue
      } else {
        if (data.is_return) {
          evntBus.$emit("set_customer_readonly", true);
          this.invoiceType = "Return";
          this.invoiceTypes = ["Return"];
        }
        this.invoice_doc = data;
        this.items = data.items;
        this.update_items_details(this.items);
        this.posa_offers = data.posa_offers || [];
        this.items.forEach((item) => {
          if (!item.posa_row_id) {
            item.posa_row_id = this.makeid(20);
          }
          if (item.batch_no) {
            this.set_batch_qty(item, item.batch_no);
          }
        });
        this.customer = data.customer;
        this.posting_date = data.posting_date || frappe.datetime.nowdate();
        this.discount_amount = data.discount_amount;
        this.additional_discount_percentage =
          data.additional_discount_percentage;
        this.items.forEach((item) => {
          if (item.serial_no) {
            item.serial_no_selected = [];
            const serial_list = item.serial_no.split("\n");
            serial_list.forEach((element) => {
              if (element.length) {
                item.serial_no_selected.push(element);
              }
            });
            item.serial_no_selected_count = item.serial_no_selected.length;
          }
        });
      }
      return old_invoice;
    },

    async new_order(data = {}) {
      let old_invoice = null;
      evntBus.$emit("set_customer_readonly", false);
      this.expanded = [];
      this.posa_offers = [];
      evntBus.$emit("set_pos_coupons", []);
      this.posa_coupons = [];
      this.return_doc = "";
      if (!data.name && !data.is_return) {
        this.items = [];
        this.customer = this.pos_profile.customer;
        this.invoice_doc = "";
        this.discount_amount = 0;
        this.additional_discount_percentage = 0;
        this.invoiceType = "Invoice";
        this.invoiceTypes = ["Invoice", "Order"];
      } else {
        if (data.is_return) {
          evntBus.$emit("set_customer_readonly", true);
          this.invoiceType = "Return";
          this.invoiceTypes = ["Return"];
        }
        this.invoice_doc = data;
        this.items = data.items;
        this.update_items_details(this.items);
        this.posa_offers = data.posa_offers || [];
        this.items.forEach((item) => {
          if (!item.posa_row_id) {
            item.posa_row_id = this.makeid(20);
          }
          if (item.batch_no) {
            this.set_batch_qty(item, item.batch_no);
          }
        });
        this.customer = data.customer;
        this.posting_date = data.posting_date || frappe.datetime.nowdate();
        this.discount_amount = data.discount_amount;
        this.additional_discount_percentage =
          data.additional_discount_percentage;
        this.items.forEach((item) => {
          if (item.serial_no) {
            item.serial_no_selected = [];
            const serial_list = item.serial_no.split("\n");
            serial_list.forEach((element) => {
              if (element.length) {
                item.serial_no_selected.push(element);
              }
            });
            item.serial_no_selected_count = item.serial_no_selected.length;
          }
        });
      }
      return old_invoice;
    },

    get_invoice_doc() {
      let doc = {};
      if (this.invoice_doc.name) {
        doc = { ...this.invoice_doc };
      }
      doc.doctype = "Sales Invoice";
      doc.is_pos = 1;
      doc.ignore_pricing_rule = 1;
      doc.company = doc.company || this.pos_profile.company;
      doc.pos_profile = doc.pos_profile || this.pos_profile.name;
      doc.campaign = doc.campaign || this.pos_profile.campaign;
      doc.currency = doc.currency || this.pos_profile.currency;
      doc.naming_series = doc.naming_series || this.pos_profile.naming_series;
      doc.customer = this.customer;
      doc.items = this.get_invoice_items();
      doc.total = this.subtotal;
      doc.discount_amount = flt(this.discount_amount);
      doc.additional_discount_percentage = flt(
        this.additional_discount_percentage
      );
      doc.posa_pos_opening_shift = this.pos_opening_shift.name;
      doc.payments = this.get_payments();
      doc.taxes = [];
      doc.is_return = this.invoice_doc.is_return;
      doc.return_against = this.invoice_doc.return_against;
      doc.posa_offers = this.posa_offers;
      doc.posa_coupons = this.posa_coupons;
      doc.posa_delivery_charges = this.selcted_delivery_charges.name;
      doc.posa_delivery_charges_rate = this.delivery_charges_rate || 0;
      doc.posting_date = this.posting_date;
      return doc;
    },

    async get_invoice_from_order_doc() {
      let doc = {};
      if (this.invoice_doc.doctype == "Sales Order") {
        await frappe.call({
          method:
            "posawesome.posawesome.api.posapp.create_sales_invoice_from_order",
          args: {
            sales_order: this.invoice_doc.name,
          },
          // async: false,
          callback: function (r) {
            if (r.message) {
              doc = r.message;
            }
          },
        });
      } else {
        doc = this.invoice_doc;
      }
      const Items = [];
      const updatedItemsData = this.get_invoice_items();
      doc.items.forEach((item) => {
        const updatedData = updatedItemsData.find(
          (updatedItem) => updatedItem.item_code === item.item_code
        );
        if (updatedData) {
          item.item_code = updatedData.item_code;
          item.posa_row_id = updatedData.posa_row_id;
          item.posa_offers = updatedData.posa_offers;
          item.posa_offer_applied = updatedData.posa_offer_applied;
          item.posa_is_offer = updatedData.posa_is_offer;
          item.posa_is_replace = updatedData.posa_is_replace;
          item.is_free_item = updatedData.is_free_item;
          item.qty = flt(updatedData.qty);
          item.rate = flt(updatedData.rate);
          item.uom = updatedData.uom;
          item.amount = flt(updatedData.qty) * flt(updatedData.rate);
          item.conversion_factor = updatedData.conversion_factor;
          item.serial_no = updatedData.serial_no;
          item.discount_percentage = flt(updatedData.discount_percentage);
          item.discount_amount = flt(updatedData.discount_amount);
          item.batch_no = updatedData.batch_no;
          item.posa_notes = updatedData.posa_notes;
          item.posa_delivery_date = updatedData.posa_delivery_date;
          item.price_list_rate = updatedData.price_list_rate;
          Items.push(item);
        }
      });

      doc.items = Items;
      const newItems = [...doc.items];
      const existingItemCodes = new Set(newItems.map((item) => item.item_code));
      updatedItemsData.forEach((updatedItem) => {
        if (!existingItemCodes.has(updatedItem.item_code)) {
          newItems.push(updatedItem);
        }
      });
      doc.items = newItems;
      doc.update_stock = 1;
      doc.is_pos = 1;
      doc.payments = this.get_payments();
      return doc;
    },

    get_invoice_items() {
      const items_list = [];
      this.items.forEach((item) => {
        const new_item = {
          sales_invoice_item: item.sales_invoice_item, // needed during returns validation in the new update ERPNext v14.83.0, Frappe v14.94.1
          item_code: item.item_code,
          item_name: item.item_name,
          posa_row_id: item.posa_row_id,
          posa_offers: item.posa_offers,
          posa_offer_applied: item.posa_offer_applied,
          posa_is_offer: item.posa_is_offer,
          posa_is_replace: item.posa_is_replace,
          is_free_item: item.is_free_item,
          qty: flt(item.qty),
          rate: flt(item.rate),
          uom: item.uom,
          amount: flt(item.qty) * flt(item.rate),
          conversion_factor: item.conversion_factor,
          serial_no: item.serial_no,
          discount_percentage: flt(item.discount_percentage),
          discount_amount: flt(item.discount_amount),
          batch_no: item.batch_no,
          posa_notes: item.posa_notes,
          posa_delivery_date: item.posa_delivery_date,
          price_list_rate: item.price_list_rate,
        };
        items_list.push(new_item);
      });

      return items_list;
    },

    get_order_items() {
      const items_list = [];
      this.items.forEach((item) => {
        const new_item = {
          item_code: item.item_code,
          posa_row_id: item.posa_row_id,
          posa_offers: item.posa_offers,
          posa_offer_applied: item.posa_offer_applied,
          posa_is_offer: item.posa_is_offer,
          posa_is_replace: item.posa_is_replace,
          is_free_item: item.is_free_item,
          qty: flt(item.qty),
          rate: flt(item.rate),
          uom: item.uom,
          amount: flt(item.qty) * flt(item.rate),
          conversion_factor: item.conversion_factor,
          serial_no: item.serial_no,
          discount_percentage: flt(item.discount_percentage),
          discount_amount: flt(item.discount_amount),
          batch_no: item.batch_no,
          posa_notes: item.posa_notes,
          posa_delivery_date: item.posa_delivery_date,
          price_list_rate: item.price_list_rate,
        };
        items_list.push(new_item);
      });

      return items_list;
    },

    get_payments() {
      const payments = [];
      this.pos_profile.payments.forEach((payment) => {
        payments.push({
          amount: 0,
          mode_of_payment: payment.mode_of_payment,
          default: payment.default,
          account: "",
        });
      });
      return payments;
    },

    update_invoice(doc) {
      const vm = this;
      frappe.call({
        method: "posawesome.posawesome.api.posapp.update_invoice",
        args: {
          data: doc,
        },
        async: false,
        callback: function (r) {
          if (r.message) {
            vm.invoice_doc = r.message;
          }
        },
      });
      return this.invoice_doc;
    },

    update_invoice_from_order(doc) {
      const vm = this;
      frappe.call({
        method: "posawesome.posawesome.api.posapp.update_invoice_from_order",
        args: {
          data: doc,
        },
        async: false,
        callback: function (r) {
          if (r.message) {
            vm.invoice_doc = r.message;
          }
        },
      });
      return this.invoice_doc;
    },

    process_invoice() {
      const doc = this.get_invoice_doc();
      //console.log("this.get_invoice_doc(): ", doc);
      if (doc.name) {
        return this.update_invoice(doc);
      }
      else if (doc.company == 'Pour Tous Distribution Center' && this.invoiceType == "Order")
        return doc;
      else {
        return this.update_invoice(doc);
      }
    },

    async process_invoice_from_order() {
      const doc = await this.get_invoice_from_order_doc();
      var up_invoice;
      if (doc.name) {
        up_invoice = await this.update_invoice_from_order(doc);
        return up_invoice;
      } else {
        return this.update_invoice_from_order(doc);
      }
    },

    /* update_order(doc) {
      const vm = this;
      frappe.call({
        method: "posawesome.posawesome.api.posapp.create_advance_sales_order",
        args: {
          data: doc,
        },
        async: false,
        callback: function (r) {
          if (r.message) {
            vm.invoice_doc = r.message;
          }
        },
      });
      return this.invoice_doc;
    }, */

    save_as_order() {
      this.invoiceType = "Order";
      this.show_payment();
    },

    async show_payment() {
      //console.log("Label-A");
      //console.log("invoice_doc: ", this.invoice_doc);
      if (!this.customer) {
        evntBus.$emit("show_mesage", {
          text: __(`There is no Customer !`),
          color: "error",
        });
        return;
      }
      if (!this.items.length) {
        evntBus.$emit("show_mesage", {
          text: __(`There are no Items !`),
          color: "error",
        });
        return;
      }
      if (!this.validate()) {
        return;
      }
      else if (this.invoice_doc.doctype == "Sales Order") {
        //console.log("Label-B");
        evntBus.$emit("show_payment", "true");
        const invoice_doc = await this.process_invoice_from_order();
        evntBus.$emit("send_invoice_doc_payment", invoice_doc);
      } else if (this.invoice_doc.doctype == "Sales Invoice") {
        //console.log("Label-C");
        // adding await below as a fix for: "TypeError: get_sales_invoice_child_table() missing 1 required positional argument: 'sales_invoice_item'"
        //const sales_invoice_item = await this.invoice_doc.items[0];
        const sales_invoice_item = this.invoice_doc.items[0];
        var sales_invoice_item_doc = {};
        frappe.call({
          method:
            "posawesome.posawesome.api.posapp.get_sales_invoice_child_table",
          args: {
            sales_invoice: this.invoice_doc.name,
            sales_invoice_item: sales_invoice_item.name,
          },
          async: false,
          callback: function (r) {
            if (r.message) {
              sales_invoice_item_doc = r.message;
            }
          },
        });
        if (sales_invoice_item_doc.sales_order) {
          //console.log("Label-D");
          evntBus.$emit("show_payment", "true");
          const invoice_doc = await this.process_invoice_from_order();
          evntBus.$emit("send_invoice_doc_payment", invoice_doc);
        } else {
          //console.log("Label-E");
          evntBus.$emit("show_payment", "true");
          const invoice_doc = this.process_invoice();
          evntBus.$emit("send_invoice_doc_payment", invoice_doc);
        }
      } else {
        //console.log("Label-F");
        evntBus.$emit("show_payment", "true");
        const invoice_doc = this.process_invoice();
        evntBus.$emit("send_invoice_doc_payment", invoice_doc);
      }
    },

    validate() {
      let value = true;
      this.items.forEach((item) => {
        if (
          this.pos_profile.posa_max_discount_allowed &&
          !item.posa_offer_applied
        ) {
          if (item.discount_amount && this.flt(item.discount_amount) > 0) {
            // calc discount percentage
            const discount_percentage =
              (this.flt(item.discount_amount) * 100) /
              this.flt(item.price_list_rate);
            if (
              discount_percentage > this.pos_profile.posa_max_discount_allowed
            ) {
              evntBus.$emit("show_mesage", {
                text: __(
                  `Discount percentage for item '{0}' cannot be greater than {1}%`,
                  [item.item_name, this.pos_profile.posa_max_discount_allowed]
                ),
                color: "error",
              });
              value = false;
            }
          }
        }
        if (this.stock_settings.allow_negative_stock != 1 || this.invoiceType != "Order") {
          if (
            this.invoiceType == "Invoice" &&
            ((item.is_stock_item && item.stock_qty && !item.actual_qty) ||
              (item.is_stock_item && item.stock_qty > item.actual_qty))
          ) {
            evntBus.$emit("show_mesage", {
              text: __(
                `The existing quantity '{0}' for item '{1}' is not enough`,
                [item.actual_qty, item.item_name]
              ),
              color: "error",
            });
            value = false;
          }
        }
        if (item.qty == 0) {
          evntBus.$emit("show_mesage", {
            text: __(`Quantity for item '{0}' cannot be Zero (0)`, [
              item.item_name,
            ]),
            color: "error",
          });
          value = false;
        }
        if (
          item.max_discount > 0 &&
          item.discount_percentage > item.max_discount
        ) {
          evntBus.$emit("show_mesage", {
            text: __(`Maximum discount for Item {0} is {1}%`, [
              item.item_name,
              item.max_discount,
            ]),
            color: "error",
          });
          value = false;
        }
        if (item.has_serial_no) {
          if (
            !this.invoice_doc.is_return &&
            (!item.serial_no_selected ||
              item.stock_qty != item.serial_no_selected.length)
          ) {
            evntBus.$emit("show_mesage", {
              text: __(`Selected serial numbers of item {0} is incorrect`, [
                item.item_name,
              ]),
              color: "error",
            });
            value = false;
          }
        }
        if (item.has_batch_no && frappe.defaults.get_user_default("company") != 'Pour Tous Distribution Center' && this.invoiceType != "Order") {
        //if (item.has_batch_no) {
          if (item.stock_qty > item.actual_batch_qty) {
            evntBus.$emit("show_mesage", {
              text: __(
                `The existing batch quantity of item {0} is not enough`,
                [item.item_name]
              ),
              color: "error",
            });
            value = false;
          }
        }
        if (this.pos_profile.posa_allow_user_to_edit_additional_discount) {
          const clac_percentage = (this.discount_amount / this.Total) * 100;
          if (clac_percentage > this.pos_profile.posa_max_discount_allowed) {
            evntBus.$emit("show_mesage", {
              text: __(`The discount should not be higher than {0}%`, [
                this.pos_profile.posa_max_discount_allowed,
              ]),
              color: "error",
            });
            value = false;
          }
        }
        if (this.invoice_doc.is_return) {
          if (this.subtotal >= 0) {
            evntBus.$emit("show_mesage", {
              text: __(`Return Invoice Total Not Correct`),
              color: "error",
            });
            value = false;
            return value;
          }
          if (Math.abs(this.subtotal) > Math.abs(this.return_doc.total)) {
            evntBus.$emit("show_mesage", {
              text: __(`Return Invoice Total should not be higher than {0}`, [
                this.return_doc.total,
              ]),
              color: "error",
            });
            value = false;
            return value;
          }
          this.items.forEach((item) => {
            const return_item = this.return_doc.items.find(
              //(element) => element.batch_no == item.batch_no //&& Math.abs(element.qty) == Math.abs(item.qty)
              (element) => ((element.item_code == item.item_code) && (element.batch_no == item.batch_no)) //&& Math.abs(element.qty) == Math.abs(item.qty)
            );
            console.log("return_item.batch_no: ", return_item.batch_no);

            if (!return_item) {
              evntBus.$emit("show_mesage", {
                text: __(
                  `The item {0} cannot be returned because it is not in the invoice {1}`,
                  [item.item_name, this.return_doc.name]
                ),
                color: "error",
              });
              value = false;
              return value;
            } else if (
              Math.abs(item.qty) > Math.abs(return_item.qty) ||
              Math.abs(item.qty) == 0
            ) {
              evntBus.$emit("show_mesage", {
                text: __(`The QTY of the item {0} cannot be greater than {1}`, [
                  item.item_name,
                  return_item.qty,
                ]),
                color: "error",
              });
              value = false;
              return value;
            }
          });
        }
      });
      return value;
    },

    get_draft_invoices() {
      const vm = this;
      frappe.call({
        method: "posawesome.posawesome.api.posapp.get_draft_invoices",
        args: {
          pos_opening_shift: this.pos_opening_shift.name,
        },
        async: false,
        callback: function (r) {
          if (r.message) {
            evntBus.$emit("open_drafts", r.message);
          }
        },
      });
    },

    open_pending_fs_bills() {
      this.display_pending_bill_details = true;
      const vm = this;
      if (this.pending_fs_bills > 0) {
        frappe.call({
          method: "posawesome.posawesome.api.posapp.open_pending_fs_bills",
          args: {
            customer: vm.customer,
            company: vm.pos_profile.company
          },
          async: false,
          callback: function (r) {
            if (r.message) {
              evntBus.$emit("open_drafts", r.message);
            }
          },
        });
      }
    },

    get_draft_orders() {
      const vm = this;
      frappe.call({
        method: "posawesome.posawesome.api.posapp.search_orders",
        args: {
          company: this.pos_profile.company,
          currency: this.pos_profile.currency,
        },
        async: false,
        callback: function (r) {
          if (r.message) {
            evntBus.$emit("open_orders", r.message);
          }
        },
      });
    },

    open_returns() {
      evntBus.$emit("open_returns", this.pos_profile.company);
    },

    close_payments() {
      evntBus.$emit("show_payment", "false");
    },

    update_items_details(items) {
      if (!items.length > 0) {
        return;
      }
      const vm = this;
      if (!vm.pos_profile) return;
      frappe.call({
        method: "posawesome.posawesome.api.posapp.get_items_details",
        async: false,
        args: {
          pos_profile: vm.pos_profile,
          items_data: items,
        },
        callback: function (r) {
          if (r.message) {
            items.forEach((item) => {
              const updated_item = r.message.find(
                (element) => element.posa_row_id == item.posa_row_id
              );
              item.actual_qty = updated_item.actual_qty;
              item.serial_no_data = updated_item.serial_no_data;
              item.batch_no_data = updated_item.batch_no_data;
              item.item_uoms = updated_item.item_uoms;
              item.has_batch_no = updated_item.has_batch_no;
              item.has_serial_no = updated_item.has_serial_no;
            });
          }
        },
      });
    },

    update_item_detail(item) {
      if (!item.item_code || this.invoice_doc.is_return) {
        return;
      }
      const vm = this;
      frappe.call({
        method: "posawesome.posawesome.api.posapp.get_item_detail",
        async: false,  // set this AJAX call to synchronous, so that the function add_item(item) get updated data when it calls update_item_detail(item)
        args: {
          warehouse: this.pos_profile.warehouse,
          doc: this.get_invoice_doc(),
          //price_list: this.pos_profile.price_list,
          price_list: this.pos_profile.selling_price_list, // the correct docfield in doctype pos_profile is selling_price_list
          item: {
            item_code: item.item_code,
            warehouse: this.pos_profile.warehouse, // adding this parameter, as it is needed at the backend, to get batch_no_data
            customer: this.customer,
            doctype: "Sales Invoice",
            name: "New Sales Invoice 1",
            company: this.pos_profile.company,
            conversion_rate: 1,
            qty: item.qty,
            price_list_rate: item.price_list_rate,
            child_docname: "New Sales Invoice Item 1",
            cost_center: this.pos_profile.cost_center,
            currency: this.pos_profile.currency,
            // plc_conversion_rate: 1,
            pos_profile: this.pos_profile.name,
            uom: item.uom,
            tax_category: "",
            transaction_type: "selling",
            update_stock: this.pos_profile.update_stock,
            price_list: this.get_price_list(),
            has_batch_no: item.has_batch_no,
            serial_no: item.serial_no,
            batch_no: item.batch_no,
            is_stock_item: item.is_stock_item,
          },
        },
        callback: function (r) {
          if (r.message) {
            const data = r.message;
            if (data.batch_no_data) {
              item.batch_no_data = data.batch_no_data;
            }
            if (
              item.has_batch_no &&
              vm.pos_profile.posa_auto_set_batch &&
              !item.batch_no &&
              data.batch_no_data
            ) {
              item.batch_no_data = data.batch_no_data;
              vm.set_batch_qty(item, item.batch_no, false);
              // item.batch_no is a dummy parameter here (it was used in previous POSA version, and may be used in other function calls of set_batch_qty)
            }
            if (data.has_pricing_rule) {
            } else if (
              vm.pos_profile.posa_apply_customer_discount &&
              vm.customer_info.posa_discount > 0 &&
              vm.customer_info.posa_discount <= 100
            ) {
              if (
                item.posa_is_offer == 0 &&
                !item.posa_is_replace &&
                item.posa_offer_applied == 0
              ) {
                if (item.max_discount > 0) {
                  item.discount_percentage =
                    item.max_discount < vm.customer_info.posa_discount
                      ? item.max_discount
                      : vm.customer_info.posa_discount;
                } else {
                  item.discount_percentage = vm.customer_info.posa_discount;
                }
              }
            }
            if (!item.batch_price) {
              if (
                !item.is_free_item &&
                !item.posa_is_offer &&
                !item.posa_is_replace
              ) {
                item.price_list_rate = data.price_list_rate;
              }
            }
            item.last_purchase_rate = data.last_purchase_rate;
            item.projected_qty = data.projected_qty;
            item.reserved_qty = data.reserved_qty;
            item.conversion_factor = data.conversion_factor;
            item.stock_qty = data.stock_qty;
            item.actual_qty = data.actual_qty;
            item.stock_uom = data.stock_uom;
            (item.has_serial_no = data.has_serial_no),
              (item.has_batch_no = data.has_batch_no),
              vm.calc_item_price(item);
          }
        },
      });
    },

    fetch_customer_details() {
      const vm = this;
      if (this.customer) {
        frappe.call({
          method: "posawesome.posawesome.api.posapp.get_customer_info",
          args: {
            customer: vm.customer,
          },
          async: false,
          callback: (r) => {
            const message = r.message;
            if (!r.exc) {
              vm.customer_info = {
                ...message,
              };
            }
            vm.update_price_list();
          },
        });
      }
    },

    get_price_list() {
      let price_list = this.pos_profile.selling_price_list;
      if (this.customer_info && this.pos_profile) {
        const { customer_price_list, customer_group_price_list } =
          this.customer_info;
        const pos_price_list = this.pos_profile.selling_price_list;
        if (customer_price_list && customer_price_list != pos_price_list) {
          price_list = customer_price_list;
        } else if (
          customer_group_price_list &&
          customer_group_price_list != pos_price_list
        ) {
          price_list = customer_group_price_list;
        }
      }
      return price_list;
    },

    // the below function updates the customer_price_list or customer_group_price_list, if they are set.
    update_price_list() {
      let price_list = this.get_price_list();
      if (price_list == this.pos_profile.selling_price_list) {
        price_list = null;
      }
      evntBus.$emit("update_customer_price_list", price_list);
    },
    update_discount_umount() {
      const value = flt(this.additional_discount_percentage);
      if (value >= -100 && value <= 100) {
        this.discount_amount = (this.Total * value) / 100;
      } else {
        this.additional_discount_percentage = 0;
        this.discount_amount = 0;
      }
    },

    calc_prices(item, value, $event) {
      if (event.target.id === "rate") {
        item.discount_percentage = 0;
        if (value < item.price_list_rate) {
          item.discount_amount = this.flt(
            this.flt(item.price_list_rate) - flt(value),
            this.currency_precision
          );
        } else if (value < 0) {
          item.rate = item.price_list_rate;
          item.discount_amount = 0;
        } else if (value > item.price_list_rate) {
          item.discount_amount = 0;
        }
      } else if (event.target.id === "discount_amount") {
        if (value < 0) {
          item.discount_amount = 0;
          item.discount_percentage = 0;
        } else {
          item.rate = flt(item.price_list_rate) - flt(value);
          item.discount_percentage = 0;
        }
      } else if (event.target.id === "discount_percentage") {
        if (value < 0) {
          item.discount_amount = 0;
          item.discount_percentage = 0;
        } else {
          item.rate = this.flt(
            flt(item.price_list_rate) -
              (flt(item.price_list_rate) * flt(value)) / 100,
            this.currency_precision
          );
          item.discount_amount = this.flt(
            flt(item.price_list_rate) - flt(+item.rate),
            this.currency_precision
          );
        }
      }
    },

    calc_item_price(item) {
      if (!item.posa_offer_applied) {
        if (item.price_list_rate) {
          item.rate = item.price_list_rate;
        }
      }
      if (item.discount_percentage) {
        item.rate =
          flt(item.price_list_rate) -
          (flt(item.price_list_rate) * flt(item.discount_percentage)) / 100;
        item.discount_amount = this.flt(
          flt(item.price_list_rate) - flt(item.rate),
          this.currency_precision
        );
      } else if (item.discount_amount) {
        item.rate = this.flt(
          flt(item.price_list_rate) - flt(item.discount_amount),
          this.currency_precision
        );
      }
    },

    calc_uom(item, value) {
      const new_uom = item.item_uoms.find((element) => element.uom == value);
      item.conversion_factor = new_uom.conversion_factor;
      if (!item.posa_offer_applied) {
        item.discount_amount = 0;
        item.discount_percentage = 0;
      }
      if (item.batch_price) {
        item.price_list_rate = item.batch_price * new_uom.conversion_factor;
      }
      this.update_item_detail(item);
    },

    QTY_textbox_update(item, qty) {
      //console.log("item.stock_qty, qty : ", item.stock_qty, qty);
      this.calc_stock_qty(item, qty);
      // in case custom_item_add_on is set, then add/subtract the add-on item
      if (item.custom_item_add_on) {
        let index = this.items.findIndex(
          (el) =>
            el.item_code === item.custom_item_add_on
        );
        let add_on_item = this.items[index];
        add_on_item.qty = qty;
        this.calc_stock_qty(add_on_item, add_on_item.qty);
      }
    },

    calc_stock_qty(item, value) {
      item.stock_qty = item.conversion_factor * value;
      //console.log("item.stock_qty: ", item.stock_qty);
    },

    set_serial_no(item) {
      if (!item.has_serial_no) return;
      item.serial_no = "";
      item.serial_no_selected.forEach((element) => {
        item.serial_no += element + "\n";
      });
      item.serial_no_selected_count = item.serial_no_selected.length;
      if (item.serial_no_selected_count != item.stock_qty) {
        item.qty = item.serial_no_selected_count;
        this.calc_stock_qty(item, item.qty);
        this.$forceUpdate();
      }
    },

    set_batch_qty(item, value, update = true) {
      const existing_items = this.items.filter(
        (element) =>
          element.item_code == item.item_code &&
          element.posa_row_id != item.posa_row_id
      );
      const used_batches = {};
      item.batch_no_data.forEach((batch) => {
        used_batches[batch.batch_no] = {
          ...batch,
          used_qty: 0,
          remaining_qty: batch.batch_qty,
        };
        existing_items.forEach((element) => {
          if (element.batch_no && element.batch_no == batch.batch_no) {
            used_batches[batch.batch_no].used_qty += element.qty;
            used_batches[batch.batch_no].remaining_qty -= element.qty;
            used_batches[batch.batch_no].batch_qty -= element.qty;
          }
        });
      });

      // set item batch_no based on:
      // 1. if batch has expiry_date we should use the batch with the nearest expiry_date
      // 2. if batch has no expiry_date we should use the batch with the earliest manufacturing_date
      // 3. we should not use batch with remaining_qty = 0
      // 4. we should the highest remaining_qty
      let batch_no_data;
      if (this.invoice_doc.is_return) { // in case of returns, also pass the batches with qty '0'
        console.log("(if) this.invoice_doc.is_return: ", this.invoice_doc.is_return);
        batch_no_data = Object.values(used_batches)
          //.filter((batch) => batch.remaining_qty > 0)
          .sort((a, b) => {
            if (a.expiry_date && b.expiry_date) {
              return a.expiry_date - b.expiry_date;
            } else if (a.expiry_date) {
              return -1;
            } else if (b.expiry_date) {
              return 1;
            } else if (a.manufacturing_date && b.manufacturing_date) {
              return a.manufacturing_date - b.manufacturing_date;
            } else if (a.manufacturing_date) {
              return -1;
            } else if (b.manufacturing_date) {
              return 1;
            } else {
              return a.remaining_qty - b.remaining_qty;
              //return b.remaining_qty - a.remaining_qty;
            }
          });
      }
      else {
        console.log("(Else) this.invoice_doc.is_return: ", this.invoice_doc.is_return);
        batch_no_data = Object.values(used_batches)
          .filter((batch) => batch.remaining_qty > 0)
          .sort((a, b) => {
            if (a.expiry_date && b.expiry_date) {
              return a.expiry_date - b.expiry_date;
            } else if (a.expiry_date) {
              return -1;
            } else if (b.expiry_date) {
              return 1;
            } else if (a.manufacturing_date && b.manufacturing_date) {
              return a.manufacturing_date - b.manufacturing_date;
            } else if (a.manufacturing_date) {
              return -1;
            } else if (b.manufacturing_date) {
              return 1;
            } else {
              return a.remaining_qty - b.remaining_qty;
              //return b.remaining_qty - a.remaining_qty;
            }
          });
      }
      if (batch_no_data.length > 0) {
        let batch_to_use = null;
        if (value) {
          batch_to_use = batch_no_data.find((batch) => batch.batch_no == value);
        }
        if (!batch_to_use) {
          //console.log("batch_no_data: ", batch_no_data);
          //console.log("item.qty: ", item.qty, " batch.remaining_qty: ", batch_no_data[0].remaining_qty);
          for (batch of batch_no_data) {
            //console.log("batch: ", batch);
            if (item.qty <= batch.remaining_qty) {
              batch_to_use = batch;
              //console.log("batch_to_use: ", batch_to_use);
              break;
            }
          }
          //batch_to_use = batch_no_data[0]; // select the batch here
        }
        item.batch_no = batch_to_use.batch_no;
        item.actual_batch_qty = batch_to_use.batch_qty;
        item.batch_no_expiry_date = batch_to_use.expiry_date;
        if (batch_to_use.batch_price) {
          item.batch_price = batch_to_use.batch_price;
          item.price_list_rate = batch_to_use.batch_price;
          item.rate = batch_to_use.batch_price;
        } else if (update) {
          item.batch_price = null;
          this.update_item_detail(item);
        }
      } else {
        item.batch_no = null;
        item.actual_batch_qty = null;
        item.batch_no_expiry_date = null;
        item.batch_price = null;
      }
      // update item batch_no_data from batch_no_data
      item.batch_no_data = batch_no_data;
    },

    /*
    bankers_rounding(value) {
      frappe.call({     // using the flt function defined in frappe.utils as it uses the banker's rounding algorithm
        method: 'posawesome.posawesome.api.posapp.bankers_rounding',
        args: {
          num: value,
          precision: this.currency_precision,
        },
        async: false,
        callback: function (r) {
          if (r.message) {
            value = r.message;
          }
        },
      });
      return value;
    },
    */

    /* shortOfflinePay(e) {
      if (e.key === "s" && (e.ctrlKey || e.metaKey)) {
        e.preventDefault();
        this.offline_fs_pay();
      }
    }, */

    shortOpenPayment(e) {
      if (e.key === "F2") {
        e.preventDefault();
        this.show_payment();
      }
    },

    shortSaveAsOrder(e) {
      if (e.key === "F4") {
        e.preventDefault();
        this.save_as_order();
      }
    },

    shortDeleteFirstItem(e) {
      if (e.key === "d" && (e.ctrlKey || e.metaKey)) {
        e.preventDefault();
        this.remove_item(this.items[0]);
      }
    },

    shortOpenFirstItem(e) {
      if (e.key === "a" && (e.ctrlKey || e.metaKey)) {
        e.preventDefault();
        this.expanded = [];
        this.expanded.push(this.items[0]);
      }
    },

    shortSelectDiscount(e) {
      if (e.key === "z" && (e.ctrlKey || e.metaKey)) {
        e.preventDefault();
        //this.$refs.discount.focus();
        this.$refs.percentage_discount.focus();
      }
    },

    makeid(length) {
      let result = "";
      const characters = "abcdefghijklmnopqrstuvwxyz0123456789";
      const charactersLength = characters.length;
      for (var i = 0; i < length; i++) {
        result += characters.charAt(
          Math.floor(Math.random() * charactersLength)
        );
      }
      return result;
    },

    checkOfferIsAppley(item, offer) {
      let applied = false;
      const item_offers = JSON.parse(item.posa_offers);
      for (const row_id of item_offers) {
        const exist_offer = this.posa_offers.find((el) => row_id == el.row_id);
        if (exist_offer && exist_offer.offer_name == offer.name) {
          applied = true;
          break;
        }
      }
      return applied;
    },

    handelOffers() {
      const offers = [];
      this.posOffers.forEach((offer) => {
        if (offer.apply_on === "Item Code") {
          const itemOffer = this.getItemOffer(offer);
          if (itemOffer) {
            offers.push(itemOffer);
          }
        } else if (offer.apply_on === "Item Group") {
          const groupOffer = this.getGroupOffer(offer);
          if (groupOffer) {
            offers.push(groupOffer);
          }
        } else if (offer.apply_on === "Brand") {
          const brandOffer = this.getBrandOffer(offer);
          if (brandOffer) {
            offers.push(brandOffer);
          }
        } else if (offer.apply_on === "Transaction") {
          const transactionOffer = this.getTransactionOffer(offer);
          if (transactionOffer) {
            offers.push(transactionOffer);
          }
        }
      });

      this.setItemGiveOffer(offers);
      this.updatePosOffers(offers);
    },

    setItemGiveOffer(offers) {
      // Set item give offer for replace
      offers.forEach((offer) => {
        if (
          offer.apply_on == "Item Code" &&
          offer.apply_type == "Item Code" &&
          offer.replace_item
        ) {
          offer.give_item = offer.item;
          offer.apply_item_code = offer.item;
        } else if (
          offer.apply_on == "Item Group" &&
          offer.apply_type == "Item Group" &&
          offer.replace_cheapest_item
        ) {
          const offerItemCode = this.getCheapestItem(offer).item_code;
          offer.give_item = offerItemCode;
          offer.apply_item_code = offerItemCode;
        }
      });
    },

    getCheapestItem(offer) {
      let itemsRowID;
      if (typeof offer.items === "string") {
        itemsRowID = JSON.parse(offer.items);
      } else {
        itemsRowID = offer.items;
      }
      const itemsList = [];
      itemsRowID.forEach((row_id) => {
        itemsList.push(this.getItemFromRowID(row_id));
      });
      const result = itemsList.reduce(function (res, obj) {
        return !obj.posa_is_replace &&
          !obj.posa_is_offer &&
          obj.price_list_rate < res.price_list_rate
          ? obj
          : res;
      });
      return result;
    },

    getItemFromRowID(row_id) {
      const item = this.items.find((el) => el.posa_row_id == row_id);
      return item;
    },

    checkQtyAnountOffer(offer, qty, amount) {
      let min_qty = false;
      let max_qty = false;
      let min_amt = false;
      let max_amt = false;
      const applys = [];

      if (offer.min_qty || offer.min_qty == 0) {
        if (qty >= offer.min_qty) {
          min_qty = true;
        }
        applys.push(min_qty);
      }

      if (offer.max_qty > 0) {
        if (qty <= offer.max_qty) {
          max_qty = true;
        }
        applys.push(max_qty);
      }

      if (offer.min_amt > 0) {
        if (amount >= offer.min_amt) {
          min_amt = true;
        }
        applys.push(min_amt);
      }

      if (offer.max_amt > 0) {
        if (amount <= offer.max_amt) {
          max_amt = true;
        }
        applys.push(max_amt);
      }
      let apply = false;
      if (!applys.includes(false)) {
        apply = true;
      }
      const res = {
        apply: apply,
        conditions: { min_qty, max_qty, min_amt, max_amt },
      };
      return res;
    },

    checkOfferCoupon(offer) {
      if (offer.coupon_based) {
        const coupon = this.posa_coupons.find(
          (el) => offer.name == el.pos_offer
        );
        if (coupon) {
          offer.coupon = coupon.coupon;
          return true;
        } else {
          return false;
        }
      } else {
        offer.coupon = null;
        return true;
      }
    },

    getItemOffer(offer) {
      let apply_offer = null;
      if (offer.apply_on === "Item Code") {
        if (this.checkOfferCoupon(offer)) {
          this.items.forEach((item) => {
            if (!item.posa_is_offer && item.item_code === offer.item) {
              const items = [];
              if (
                offer.offer === "Item Price" &&
                item.posa_offer_applied &&
                !this.checkOfferIsAppley(item, offer)
              ) {
              } else {
                const res = this.checkQtyAnountOffer(
                  offer,
                  item.stock_qty,
                  item.stock_qty * item.price_list_rate
                );
                if (res.apply) {
                  items.push(item.posa_row_id);
                  offer.items = items;
                  apply_offer = offer;
                }
              }
            }
          });
        }
      }
      return apply_offer;
    },

    getGroupOffer(offer) {
      let apply_offer = null;
      if (offer.apply_on === "Item Group") {
        if (this.checkOfferCoupon(offer)) {
          const items = [];
          let total_count = 0;
          let total_amount = 0;
          this.items.forEach((item) => {
            if (!item.posa_is_offer && item.item_group === offer.item_group) {
              if (
                offer.offer === "Item Price" &&
                item.posa_offer_applied &&
                !this.checkOfferIsAppley(item, offer)
              ) {
              } else {
                total_count += item.stock_qty;
                total_amount += item.stock_qty * item.price_list_rate;
                items.push(item.posa_row_id);
              }
            }
          });
          if (total_count || total_amount) {
            const res = this.checkQtyAnountOffer(
              offer,
              total_count,
              total_amount
            );
            if (res.apply) {
              offer.items = items;
              apply_offer = offer;
            }
          }
        }
      }
      return apply_offer;
    },

    getBrandOffer(offer) {
      let apply_offer = null;
      if (offer.apply_on === "Brand") {
        if (this.checkOfferCoupon(offer)) {
          const items = [];
          let total_count = 0;
          let total_amount = 0;
          this.items.forEach((item) => {
            if (!item.posa_is_offer && item.brand === offer.brand) {
              if (
                offer.offer === "Item Price" &&
                item.posa_offer_applied &&
                !this.checkOfferIsAppley(item, offer)
              ) {
              } else {
                total_count += item.stock_qty;
                total_amount += item.stock_qty * item.price_list_rate;
                items.push(item.posa_row_id);
              }
            }
          });
          if (total_count || total_amount) {
            const res = this.checkQtyAnountOffer(
              offer,
              total_count,
              total_amount
            );
            if (res.apply) {
              offer.items = items;
              apply_offer = offer;
            }
          }
        }
      }
      return apply_offer;
    },
    getTransactionOffer(offer) {
      let apply_offer = null;
      if (offer.apply_on === "Transaction") {
        if (this.checkOfferCoupon(offer)) {
          let total_qty = 0;
          this.items.forEach((item) => {
            if (!item.posa_is_offer && !item.posa_is_replace) {
              total_qty += item.stock_qty;
            }
          });
          const items = [];
          const total_count = total_qty;
          const total_amount = this.Total;
          if (total_count || total_amount) {
            const res = this.checkQtyAnountOffer(
              offer,
              total_count,
              total_amount
            );
            if (res.apply) {
              this.items.forEach((item) => {
                items.push(item.posa_row_id);
              });
              offer.items = items;
              apply_offer = offer;
            }
          }
        }
      }
      return apply_offer;
    },

    updatePosOffers(offers) {
      evntBus.$emit("update_pos_offers", offers);
    },

    updateInvoiceOffers(offers) {
      this.posa_offers.forEach((invoiceOffer) => {
        const existOffer = offers.find(
          (offer) => invoiceOffer.row_id == offer.row_id
        );
        if (!existOffer) {
          this.removeApplyOffer(invoiceOffer);
        }
      });
      offers.forEach((offer) => {
        const existOffer = this.posa_offers.find(
          (invoiceOffer) => invoiceOffer.row_id == offer.row_id
        );
        if (existOffer) {
          existOffer.items = JSON.stringify(offer.items);
          if (
            existOffer.offer === "Give Product" &&
            existOffer.give_item &&
            existOffer.give_item != offer.give_item
          ) {
            const item_to_remove = this.items.find(
              (item) => item.posa_row_id == existOffer.give_item_row_id
            );
            if (item_to_remove) {
              const updated_item_offers = offer.items.filter(
                (row_id) => row_id != item_to_remove.posa_row_id
              );
              offer.items = updated_item_offers;
              this.remove_item(item_to_remove);
              existOffer.give_item_row_id = null;
              existOffer.give_item = null;
            }
            const newItemOffer = this.ApplyOnGiveProduct(offer);
            if (offer.replace_cheapest_item) {
              const cheapestItem = this.getCheapestItem(offer);
              const oldBaseItem = this.items.find(
                (el) => el.posa_row_id == item_to_remove.posa_is_replace
              );
              newItemOffer.qty = item_to_remove.qty;
              if (oldBaseItem && !oldBaseItem.posa_is_replace) {
                oldBaseItem.qty += item_to_remove.qty;
              } else {
                const restoredItem = this.ApplyOnGiveProduct(
                  {
                    given_qty: item_to_remove.qty,
                  },
                  item_to_remove.item_code
                );
                restoredItem.posa_is_offer = 0;
                this.items.unshift(restoredItem);
              }
              newItemOffer.posa_is_offer = 0;
              newItemOffer.posa_is_replace = cheapestItem.posa_row_id;
              const diffQty = cheapestItem.qty - newItemOffer.qty;
              if (diffQty <= 0) {
                newItemOffer.qty += diffQty;
                this.remove_item(cheapestItem);
                newItemOffer.posa_row_id = cheapestItem.posa_row_id;
                newItemOffer.posa_is_replace = newItemOffer.posa_row_id;
              } else {
                cheapestItem.qty = diffQty;
              }
            }
            this.items.unshift(newItemOffer);
            existOffer.give_item_row_id = newItemOffer.posa_row_id;
            existOffer.give_item = newItemOffer.item_code;
          } else if (
            existOffer.offer === "Give Product" &&
            existOffer.give_item &&
            existOffer.give_item == offer.give_item &&
            (offer.replace_item || offer.replace_cheapest_item)
          ) {
            this.$nextTick(function () {
              const offerItem = this.getItemFromRowID(
                existOffer.give_item_row_id
              );
              const diff = offer.given_qty - offerItem.qty;
              if (diff > 0) {
                const itemsRowID = JSON.parse(existOffer.items);
                const itemsList = [];
                itemsRowID.forEach((row_id) => {
                  itemsList.push(this.getItemFromRowID(row_id));
                });
                const existItem = itemsList.find(
                  (el) =>
                    el.item_code == offerItem.item_code &&
                    el.posa_is_replace != offerItem.posa_row_id
                );
                if (existItem) {
                  const diffExistQty = existItem.qty - diff;
                  if (diffExistQty > 0) {
                    offerItem.qty += diff;
                    existItem.qty -= diff;
                  } else {
                    offerItem.qty += existItem.qty;
                    this.remove_item(existItem);
                  }
                }
              }
            });
          } else if (existOffer.offer === "Item Price") {
            this.ApplyOnPrice(offer);
          } else if (existOffer.offer === "Grand Total") {
            this.ApplyOnTotal(offer);
          }
          this.addOfferToItems(existOffer);
        } else {
          this.applyNewOffer(offer);
        }
      });
    },

    removeApplyOffer(invoiceOffer) {
      if (invoiceOffer.offer === "Item Price") {
        this.RemoveOnPrice(invoiceOffer);
        const index = this.posa_offers.findIndex(
          (el) => el.row_id === invoiceOffer.row_id
        );
        this.posa_offers.splice(index, 1);
      }
      if (invoiceOffer.offer === "Give Product") {
        const item_to_remove = this.items.find(
          (item) => item.posa_row_id == invoiceOffer.give_item_row_id
        );
        const index = this.posa_offers.findIndex(
          (el) => el.row_id === invoiceOffer.row_id
        );
        this.posa_offers.splice(index, 1);
        this.remove_item(item_to_remove);
      }
      if (invoiceOffer.offer === "Grand Total") {
        this.RemoveOnTotal(invoiceOffer);
        const index = this.posa_offers.findIndex(
          (el) => el.row_id === invoiceOffer.row_id
        );
        this.posa_offers.splice(index, 1);
      }
      if (invoiceOffer.offer === "Loyalty Point") {
        const index = this.posa_offers.findIndex(
          (el) => el.row_id === invoiceOffer.row_id
        );
        this.posa_offers.splice(index, 1);
      }
      this.deleteOfferFromItems(invoiceOffer);
    },

    applyNewOffer(offer) {
      if (offer.offer === "Item Price") {
        this.ApplyOnPrice(offer);
      }
      if (offer.offer === "Give Product") {
        let itemsRowID;
        if (typeof offer.items === "string") {
          itemsRowID = JSON.parse(offer.items);
        } else {
          itemsRowID = offer.items;
        }
        if (
          offer.apply_on == "Item Code" &&
          offer.apply_type == "Item Code" &&
          offer.replace_item
        ) {
          const item = this.ApplyOnGiveProduct(offer, offer.item);
          item.posa_is_replace = itemsRowID[0];
          const baseItem = this.items.find(
            (el) => el.posa_row_id == item.posa_is_replace
          );
          const diffQty = baseItem.qty - offer.given_qty;
          item.posa_is_offer = 0;
          if (diffQty <= 0) {
            item.qty = baseItem.qty;
            this.remove_item(baseItem);
            item.posa_row_id = item.posa_is_replace;
          } else {
            baseItem.qty = diffQty;
          }
          this.items.unshift(item);
          offer.give_item_row_id = item.posa_row_id;
        } else if (
          offer.apply_on == "Item Group" &&
          offer.apply_type == "Item Group" &&
          offer.replace_cheapest_item
        ) {
          const itemsList = [];
          itemsRowID.forEach((row_id) => {
            itemsList.push(this.getItemFromRowID(row_id));
          });
          const baseItem = itemsList.find(
            (el) => el.item_code == offer.give_item
          );
          const item = this.ApplyOnGiveProduct(offer, offer.give_item);
          item.posa_is_offer = 0;
          item.posa_is_replace = baseItem.posa_row_id;
          const diffQty = baseItem.qty - offer.given_qty;
          if (diffQty <= 0) {
            item.qty = baseItem.qty;
            this.remove_item(baseItem);
            item.posa_row_id = item.posa_is_replace;
          } else {
            baseItem.qty = diffQty;
          }
          this.items.unshift(item);
          offer.give_item_row_id = item.posa_row_id;
        } else {
          const item = this.ApplyOnGiveProduct(offer);
          this.items.unshift(item);
          if (item) {
            offer.give_item_row_id = item.posa_row_id;
          }
        }
      }
      if (offer.offer === "Grand Total") {
        this.ApplyOnTotal(offer);
      }
      if (offer.offer === "Loyalty Point") {
        evntBus.$emit("show_mesage", {
          text: __("Loyalty Point Offer Applied"),
          color: "success",
        });
      }

      const newOffer = {
        offer_name: offer.name,
        row_id: offer.row_id,
        apply_on: offer.apply_on,
        offer: offer.offer,
        items: JSON.stringify(offer.items),
        give_item: offer.give_item,
        give_item_row_id: offer.give_item_row_id,
        offer_applied: offer.offer_applied,
        coupon_based: offer.coupon_based,
        coupon: offer.coupon,
      };
      this.posa_offers.push(newOffer);
      this.addOfferToItems(newOffer);
    },

    ApplyOnGiveProduct(offer, item_code) {
      if (!item_code) {
        item_code = offer.give_item;
      }
      const items = this.allItems;
      const item = items.find((item) => item.item_code == item_code);
      if (!item) {
        return;
      }
      const new_item = { ...item };
      new_item.qty = offer.given_qty;
      new_item.stock_qty = offer.given_qty;
      new_item.rate = offer.discount_type === "Rate" ? offer.rate : item.rate;
      new_item.discount_amount =
        offer.discount_type === "Discount Amount" ? offer.discount_amount : 0;
      new_item.discount_percentage =
        offer.discount_type === "Discount Percentage"
          ? offer.discount_percentage
          : 0;
      new_item.discount_amount_per_item = 0;
      new_item.uom = item.uom ? item.uom : item.stock_uom;
      new_item.actual_batch_qty = "";
      new_item.conversion_factor = 1;
      new_item.posa_offers = JSON.stringify([]);
      new_item.posa_offer_applied = 0;
      new_item.posa_is_offer = 1;
      new_item.posa_is_replace = null;
      new_item.posa_notes = "";
      new_item.posa_delivery_date = "";
      new_item.is_free_item =
        (offer.discount_type === "Rate" && !offer.rate) ||
        (offer.discount_type === "Discount Percentage" &&
          offer.discount_percentage == 0)
          ? 1
          : 0;
      new_item.posa_row_id = this.makeid(20);
      new_item.price_list_rate =
        (offer.discount_type === "Rate" && !offer.rate) ||
        (offer.discount_type === "Discount Percentage" &&
          offer.discount_percentage == 0)
          ? 0
          : item.rate;
      if (
        (!this.pos_profile.posa_auto_set_batch && new_item.has_batch_no) ||
        new_item.has_serial_no
      ) {
        this.expanded.push(new_item);
      }
      this.update_item_detail(new_item);
      return new_item;
    },

    ApplyOnPrice(offer) {
      this.items.forEach((item) => {
        if (offer.items.includes(item.posa_row_id)) {
          const item_offers = JSON.parse(item.posa_offers);
          if (!item_offers.includes(offer.row_id)) {
            if (offer.discount_type === "Rate") {
              item.rate = offer.rate;
            } else if (offer.discount_type === "Discount Percentage") {
              item.discount_percentage += offer.discount_percentage;
            } else if (offer.discount_type === "Discount Amount") {
              item.discount_amount += offer.discount_amount;
            }
            item.posa_offer_applied = 1;
            this.calc_item_price(item);
          }
        }
      });
    },

    RemoveOnPrice(offer) {
      this.items.forEach((item) => {
        const item_offers = JSON.parse(item.posa_offers);
        if (item_offers.includes(offer.row_id)) {
          const originalOffer = this.posOffers.find(
            (el) => el.name == offer.offer_name
          );
          if (originalOffer) {
            if (originalOffer.discount_type === "Rate") {
              item.rate = item.price_list_rate;
            } else if (originalOffer.discount_type === "Discount Percentage") {
              item.discount_percentage -= offer.discount_percentage;
              if (!item.discount_percentage) {
                item.discount_percentage = 0;
                item.discount_amount = 0;
                item.rate = item.price_list_rate;
              }
            } else if (originalOffer.discount_type === "Discount Amount") {
              item.discount_amount -= offer.discount_amount;
            }
            this.calc_item_price(item);
          }
        }
      });
    },

    ApplyOnTotal(offer) {
      if (!offer.name) {
        offer = this.posOffers.find((el) => el.name == offer.offer_name);
      }
      if (
        (!this.discount_percentage_offer_name ||
          this.discount_percentage_offer_name == offer.name) &&
        offer.discount_percentage > 0 &&
        offer.discount_percentage <= 100
      ) {
        this.discount_amount = this.flt(
          (flt(this.Total) * flt(offer.discount_percentage)) / 100,
          this.currency_precision
        );
        this.discount_percentage_offer_name = offer.name;
      }
    },

    RemoveOnTotal(offer) {
      if (
        this.discount_percentage_offer_name &&
        this.discount_percentage_offer_name == offer.offer_name
      ) {
        this.discount_amount = 0;
        this.discount_percentage_offer_name = null;
      }
    },

    addOfferToItems(offer) {
      const offer_items = JSON.parse(offer.items);
      offer_items.forEach((el) => {
        this.items.forEach((exist_item) => {
          if (exist_item.posa_row_id == el) {
            const item_offers = JSON.parse(exist_item.posa_offers);
            if (!item_offers.includes(offer.row_id)) {
              item_offers.push(offer.row_id);
              if (offer.offer === "Item Price") {
                exist_item.posa_offer_applied = 1;
              }
            }
            exist_item.posa_offers = JSON.stringify(item_offers);
          }
        });
      });
    },

    deleteOfferFromItems(offer) {
      const offer_items = JSON.parse(offer.items);
      offer_items.forEach((el) => {
        this.items.forEach((exist_item) => {
          if (exist_item.posa_row_id == el) {
            const item_offers = JSON.parse(exist_item.posa_offers);
            const updated_item_offers = item_offers.filter(
              (row_id) => row_id != offer.row_id
            );
            if (offer.offer === "Item Price") {
              exist_item.posa_offer_applied = 0;
            }
            exist_item.posa_offers = JSON.stringify(updated_item_offers);
          }
        });
      });
    },

    validate_due_date(item) {
      const today = frappe.datetime.now_date();
      const parse_today = Date.parse(today);
      const new_date = Date.parse(item.posa_delivery_date);
      if (new_date < parse_today) {
        setTimeout(() => {
          item.posa_delivery_date = today;
        }, 0);
      }
    },
    load_print_page(invoice_name) {
      const print_format =
        this.pos_profile.print_format_for_online ||
        this.pos_profile.print_format;
      const letter_head = this.pos_profile.letter_head || 0;
      const url =
        frappe.urllib.get_base_url() +
        "/printview?doctype=Sales%20Invoice&name=" +
        invoice_name +
        "&trigger_print=1" +
        "&format=" +
        print_format +
        "&no_letterhead=" +
        letter_head;
      const printWindow = window.open(url, "Print");
      printWindow.addEventListener(
        "load",
        function () {
          printWindow.print();
          // printWindow.close();
          // NOTE : uncomoent this to auto closing printing window
        },
        true
      );
    },

    print_draft_invoice() {
      if (!this.pos_profile.posa_allow_print_draft_invoices) {
        evntBus.$emit("show_mesage", {
          text: __(`You are not allowed to print draft invoices`),
          color: "error",
        });
        return;
      }
      let invoice_name = this.invoice_doc.name;
      frappe.run_serially([
        () => {
          const invoice_doc = this.new_invoice();
          invoice_name = invoice_doc.name ? invoice_doc.name : invoice_name;
        },
        () => {
          this.load_print_page(invoice_name);
        },
      ]);
    },
    set_delivery_charges() {
      const vm = this;
      if (
        !this.pos_profile ||
        !this.customer ||
        !this.pos_profile.posa_use_delivery_charges
      ) {
        this.delivery_charges = [];
        this.delivery_charges_rate = 0;
        this.selcted_delivery_charges = {};
        return;
      }
      this.delivery_charges_rate = 0;
      this.selcted_delivery_charges = {};
      frappe.call({
        method:
          "posawesome.posawesome.api.posapp.get_applicable_delivery_charges",
        args: {
          company: this.pos_profile.company,
          pos_profile: this.pos_profile.name,
          customer: this.customer,
        },
        async: true,
        callback: function (r) {
          if (r.message) {
            vm.delivery_charges = r.message;
          }
        },
      });
    },
    deliveryChargesFilter(item, queryText, itemText) {
      const textOne = item.name.toLowerCase();
      const searchText = queryText.toLowerCase();
      return textOne.indexOf(searchText) > -1;
    },
    update_delivery_charges() {
      if (this.selcted_delivery_charges) {
        this.delivery_charges_rate = this.selcted_delivery_charges.rate;
      } else {
        this.delivery_charges_rate = 0;
      }
    },
  },

  mounted() {
    evntBus.$on("register_pos_profile", (data) => {
      this.pos_profile = data.pos_profile;
      this.customer = data.pos_profile.customer;
      this.pos_opening_shift = data.pos_opening_shift;
      this.stock_settings = data.stock_settings;
      this.float_precision =
        frappe.defaults.get_default("float_precision") || 2;
      this.currency_precision =
        frappe.defaults.get_default("currency_precision") || 2;
      this.rounding_method = 
        frappe.defaults.get_default("rounding_method");
      this.invoiceType = this.pos_profile.posa_default_sales_order
        ? "Order"
        : "Invoice";
    });
    evntBus.$on("checkout", () => {
      this.$refs.checkout.$el.focus();
    });
    evntBus.$on("add_item", (item) => {
      this.add_item(item);
    });
    evntBus.$on("update_customer", (customer) => {
      this.customer = customer;
      if (customer && this.pos_profile.posa_enable_fs_payments) {
        if (!this.fs_offline) {
          this.fs_balance_check(customer);
          this.pending_fs_bills_check(customer);
        }
      }
      if (customer && this.pos_profile.posa_allow_sales_order && frappe.defaults.get_user_default("company") != 'Pour Tous Distribution Center')
        this.get_customer_type(customer); // for setting "Sales Orders" for B2B customers, with customer_type as "company"
    });
    evntBus.$on("reset_fs_variables", () => {
      this.reset_fs_variables();
    });
    evntBus.$on("fetch_customer_details", () => {
      this.fetch_customer_details();
    });
    evntBus.$on("new_invoice", () => {
      this.reset_fs_variables();
      this.invoice_doc = "";
      this.cancel_invoice();
    });
    evntBus.$on("load_invoice", (data) => {
      //console.log("data: ", data);
      this.new_invoice(data);

      if (this.invoice_doc.is_return) {
        this.discount_amount = -data.discount_amount;
        this.additional_discount_percentage =
          -data.additional_discount_percentage;
        //this.return_doc = data;
        const vm = this;
        frappe.call({
          method:
            "posawesome.posawesome.api.posapp.get_return_doc",
          args: {
            return_doc: this.invoice_doc.return_against,
          },
          async: false,
          callback: function (r) {
            if (r.message) {
              vm.return_doc = r.message;
            }
          },
        });
        //console.log("this.return_doc: ", this.return_doc);
      } else {
        evntBus.$emit("set_pos_coupons", data.posa_coupons);
      }
    });
    evntBus.$on("load_order", (data) => {
      this.new_order(data);
      // evntBus.$emit("set_pos_coupons", data.posa_coupons);
    });
    evntBus.$on("set_offers", (data) => {
      this.posOffers = data;
    });
    evntBus.$on("update_invoice_offers", (data) => {
      this.updateInvoiceOffers(data);
    });
    evntBus.$on("update_invoice_coupons", (data) => {
      this.posa_coupons = data;
      this.handelOffers();
    });
    evntBus.$on("set_all_items", (data) => {
      this.allItems = data;
      this.items.forEach((item) => {
        this.update_item_detail(item);
      });
    });
    evntBus.$on("load_return_invoice", (data) => {
      this.new_invoice(data.invoice_doc);
      this.discount_amount = -data.return_doc.discount_amount;
      this.additional_discount_percentage =
        -data.return_doc.additional_discount_percentage;
      this.return_doc = data.return_doc;
    });
    evntBus.$on("set_new_line", (data) => {
      this.new_line = data;
    });
  },
  beforeDestroy() {
    evntBus.$off("register_pos_profile");
    evntBus.$off("add_item");
    evntBus.$off("update_customer");
    evntBus.$off("fetch_customer_details");
    evntBus.$off("new_invoice");
    evntBus.$off("set_offers");
    evntBus.$off("update_invoice_offers");
    evntBus.$off("update_invoice_coupons");
    evntBus.$off("set_all_items");
    evntBus.$off("reset_fs_variables");
  },
  created() {
    document.addEventListener("keydown", this.shortOpenPayment.bind(this));
    document.addEventListener("keydown", this.shortSaveAsOrder.bind(this));
    document.addEventListener("keydown", this.shortDeleteFirstItem.bind(this));
    document.addEventListener("keydown", this.shortOpenFirstItem.bind(this));
    document.addEventListener("keydown", this.shortSelectDiscount.bind(this));
    //document.addEventListener("keydown", this.shortOfflinePay.bind(this));
  },
  destroyed() {
    document.removeEventListener("keydown", this.shortOpenPayment);
    document.removeEventListener("keydown", this.shortSaveAsOrder);
    document.removeEventListener("keydown", this.shortDeleteFirstItem);
    document.removeEventListener("keydown", this.shortOpenFirstItem);
    document.removeEventListener("keydown", this.shortSelectDiscount);
    //document.removeEventListener("keydown", this.shortOfflinePay);
  },
  watch: {
    subtotal() {
      // watch only when customer is set; exclude returns; exclude cases where balance_available is 'not yet set (null)' or is '-1'
      if (this.customer && this.balance_available != null && this.balance_available >= 0 && this.subtotal >= 0) {
        if (this.subtotal > this.balance_available) {
          this.dynamic_fs_balance_color = 'error';
          this.dynamic_fs_balance_icon = 'mdi-bank';
          evntBus.$emit('show_mesage', {
            text: 'Insufficient Balance',
            color: 'warning',
          });
        }
        else if (this.subtotal < this.balance_available || (this.subtotal == 0 && this.balance_available > 0)) {
          this.dynamic_fs_balance_color = 'success';
          this.dynamic_fs_balance_icon = 'mdi-bank';
        }
      }
    },
    customer() {
      this.close_payments();
      evntBus.$emit("set_customer", this.customer);
      this.fetch_customer_details();
      this.set_delivery_charges();
    },
    customer_info() {
      evntBus.$emit("set_customer_info_to_edit", this.customer_info);
    },
    expanded(data_value) {
      // this.update_items_details(data_value);
      if (data_value.length > 0) {
        this.update_item_detail(data_value[0]);
      }
    },
    discount_percentage_offer_name() {
      evntBus.$emit("update_discount_percentage_offer_name", {
        value: this.discount_percentage_offer_name,
      });
    },
    items: {
      deep: true,
      handler(items) {
        this.handelOffers();
        this.$forceUpdate();
      },
    },
    invoiceType() {
      evntBus.$emit("update_invoice_type", this.invoiceType);
    },
    discount_amount() {
      if (!this.discount_amount || this.discount_amount == 0) {
        this.additional_discount_percentage = 0;
      } else if (this.pos_profile.posa_use_percentage_discount) {
        this.additional_discount_percentage =
          (this.discount_amount / this.Total) * 100;
      } else {
        this.additional_discount_percentage = 0;
      }
    },
  },
};
</script>

<style scoped>
.border_line_bottom {
  border-bottom: 1px solid lightgray;
}
.disable-events {
  pointer-events: none;
}
</style>
