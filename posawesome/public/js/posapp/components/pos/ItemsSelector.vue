<template>
  <div>
    <v-card
      class="selection mx-auto grey lighten-5 mt-3"
      style="max-height: 75vh; height: 75vh"
    >
      <v-progress-linear
        :active="loading"
        :indeterminate="loading"
        absolute
        top
        color="info"
      ></v-progress-linear>
      <v-row class="items px-2 py-1" no-gutters>
        <v-col class="pb-0 mb-2">
          <v-text-field
            dense
            clearable
            outlined
            color="primary"
            :label="frappe._('Search Item Name / Code')"
            hint="Search by item code, serial number, batch no or barcode"
            background-color="white"
            hide-details
            v-model="debounce_search"
            @keydown.esc="esc_event"
            @keydown.enter="search_onchange"
            @keydown.right="checkout"
            ref="debounce_search"
          ></v-text-field>
        </v-col>
        <v-col cols="3" class="pb-0 mb-2" v-if="pos_profile.posa_input_qty">
          <v-text-field
            dense
            outlined
            color="primary"
            :label="frappe._('QTY')"
            background-color="white"
            hide-details
            v-model.number="qty"
            @keydown.esc="esc_event"
            @keydown.enter="enter_event"
            @keydown.right="scale_button"
            ref="input_qty"
          ></v-text-field>
        </v-col>
        <v-col
          cols="1"
          v-if="pos_profile.posa_input_qty && pos_profile.posa_input_weighing_scale"
          align="center"
        >
          <v-btn
            icon
            text
            color="teal darken-2"
            @click="get_weight_from_scale"
            @keydown.esc="esc_event"
            ref="weighing_scale"
          >
            wt
            <v-icon>mdi-scale</v-icon>
          </v-btn>
        </v-col>
        <v-col cols="2" class="pb-0 mb-2" v-if="pos_profile.posa_new_line">
          <v-checkbox
            v-model="new_line"
            color="accent"
            value="true"
            label="NLine"
            dense
            hide-details
          ></v-checkbox>
        </v-col>
        <v-col cols="12" class="pt-0 mt-0">
          <div fluid class="items" v-if="items_view == 'card'">
            <v-row dense class="overflow-y-auto" style="max-height: 67vh">
              <v-col
                v-for="(item, idx) in filtred_items"
                :key="idx"
                xl="2"
                lg="3"
                md="6"
                sm="6"
                cols="6"
                min-height="50"
              >
                <v-card hover="hover" @click="add_item(item)">
                  <v-img
                    :src="
                      item.image ||
                      '/assets/posawesome/js/posapp/components/pos/placeholder-image.png'
                    "
                    class="white--text align-end"
                    gradient="to bottom, rgba(0,0,0,0), rgba(0,0,0,0.4)"
                    height="100px"
                  >
                    <v-card-text
                      v-text="item.item_code + ': ' + item.item_name"
                      class="text-caption px-1 pb-0"
                    ></v-card-text>
                  </v-img>
                  <v-card-text class="text--primary pa-1">
                    <div class="text-caption primary--text">
                      {{ currencySymbol(item.currency) || "" }}
                      {{ formtCurrency(item.rate) || 0 }}
                    </div>
                    <div class="text-caption golden--text">
                      {{ formtFloat(item.actual_qty) || 0 }}
                      {{ item.stock_uom || "" }}
                    </div>
                  </v-card-text>
                </v-card>
              </v-col>
            </v-row>
          </div>
          <div fluid class="items" v-if="items_view == 'list'">
            <div class="my-0 py-0 overflow-y-auto" style="max-height: 65vh">
              <template>
                <v-data-table
                  :headers="getItmesHeaders()"
                  :items="filtred_items"
                  item-key="item_code"
                  class="elevation-1"
                  :items-per-page="itemsPerPage"
                  hide-default-footer
                  @click:row="click_add_item"
                >
                  <template v-slot:item.rate="{ item }">
                    <span class="primary--text"
                      >{{ currencySymbol(item.currency) }}
                      {{ formtCurrency(item.rate) }}</span
                    >
                  </template>
                  <template v-slot:item.actual_qty="{ item }">
                    <span class="golden--text">{{
                      formtFloat(item.actual_qty)
                    }}</span>
                  </template>
                </v-data-table>
              </template>
            </div>
          </div>
        </v-col>
      </v-row>
    </v-card>
    <v-card class="cards mb-0 mt-3 pa-2 grey lighten-5">
      <v-row no-gutters align="center" justify="center">
        <v-col cols="12">
          <v-select
            :items="items_group"
            :label="frappe._('Items Group')"
            dense
            outlined
            hide-details
            v-model="item_group"
            v-on:change="search_onchange"
          ></v-select>
        </v-col>
        <v-col cols="3" class="mt-1">
          <v-btn-toggle
            v-model="items_view"
            color="primary"
            group
            dense
            rounded
          >
            <v-btn small value="list">{{ __("List") }}</v-btn>
            <v-btn small value="card">{{ __("Card") }}</v-btn>
          </v-btn-toggle>
        </v-col>
        <v-col cols="4" class="mt-2">
          <v-btn small block color="primary" text @click="show_coupons"
            >{{ couponsCount }} {{ __("Coupons") }}</v-btn
          >
        </v-col>
        <v-col cols="5" class="mt-2">
          <v-btn small block color="primary" text @click="show_offers"
            >{{ offersCount }} {{ __("Offers") }} : {{ appliedOffersCount }}
            {{ __("Applied") }}</v-btn
          >
        </v-col>
      </v-row>
    </v-card>
  </div>
</template>

<script>
import { evntBus } from "../../bus";
import format from "../../format";
import _ from "lodash";
export default {
  mixins: [format],
  data: () => ({
    pos_profile: "",
    flags: {},
    items_view: "list",
    item_group: "ALL",
    loading: false,
    items_group: ["ALL"],
    items: [],
    search: "",
    first_search: "",
    itemsPerPage: 1000,
    offersCount: 0,
    appliedOffersCount: 0,
    couponsCount: 0,
    appliedCouponsCount: 0,
    customer_price_list: null,
    customer: null,
    new_line: false,
    qty: 1,
    scale_port_promise: '',
  }),

  watch: {
    filtred_items(new_value, old_value) {
      if (!this.pos_profile.pose_use_limit_search) {
        if (new_value.length != old_value.length) {
          this.update_items_details(new_value);
        }
      }
    },
    customer() {
      this.get_items();
    },
    new_line() {
      evntBus.$emit("set_new_line", this.new_line);
    },
  },

  methods: {
    show_offers() {
      evntBus.$emit("show_offers", "true");
    },
    show_coupons() {
      evntBus.$emit("show_coupons", "true");
    },
    get_items() {
      if (!this.pos_profile) {
        console.error("No POS Profile");
        return;
      }
      const vm = this;
      this.loading = true;
      let search = this.get_search(this.first_search);
      let gr = "";
      let sr = "";
      if (search) {
        sr = search;
      }
      if (vm.item_group != "ALL") {
        gr = vm.item_group.toLowerCase();
      }
      if (
        vm.pos_profile.posa_local_storage &&
        localStorage.items_storage &&
        !vm.pos_profile.pose_use_limit_search
      ) {
        vm.items = JSON.parse(localStorage.getItem("items_storage"));
        evntBus.$emit("set_all_items", vm.items);
        vm.loading = false;
      }
      frappe.call({
        method: "posawesome.posawesome.api.posapp.get_items",
        args: {
          pos_profile: vm.pos_profile,
          price_list: vm.customer_price_list,
          item_group: gr,
          search_value: sr,
          customer: vm.customer,
        },
        callback: function (r) {
          if (r.message) {
            vm.items = r.message;
            evntBus.$emit("set_all_items", vm.items);
            vm.loading = false;
            console.info("Items Loaded");
            if (
              vm.pos_profile.posa_local_storage &&
              !vm.pos_profile.pose_use_limit_search
            ) {
              localStorage.setItem("items_storage", "");
              try {
                localStorage.setItem(
                  "items_storage",
                  JSON.stringify(r.message)
                );
              } catch (e) {
                console.error(e);
              }
            }
            if (vm.pos_profile.pose_use_limit_search) {
              vm.enter_event();
            }
          }
        },
      });
    },
    get_items_groups() {
      if (!this.pos_profile) {
        console.log("No POS Profile");
        return;
      }
      if (this.pos_profile.item_groups.length > 0) {
        this.pos_profile.item_groups.forEach((element) => {
          if (element.item_group !== "All Item Groups") {
            this.items_group.push(element.item_group);
          }
        });
      } else {
        const vm = this;
        frappe.call({
          method: "posawesome.posawesome.api.posapp.get_items_groups",
          args: {},
          callback: function (r) {
            if (r.message) {
              r.message.forEach((element) => {
                vm.items_group.push(element.name);
              });
            }
          },
        });
      }
    },
    getItmesHeaders() {
      const items_headers = [
        {
          text: __("Name"),
          align: "start",
          sortable: true,
          value: "item_name",
        },
        {
          text: __("Code"),
          align: "start",
          sortable: true,
          value: "item_code",
        },
        { text: __("Rate"), value: "rate", align: "start" },
        { text: __("Available QTY"), value: "actual_qty", align: "start" },
        { text: __("UOM"), value: "stock_uom", align: "start" },
      ];
      if (!this.pos_profile.posa_display_item_code) {
        items_headers.splice(1, 1);
      }

      return items_headers;
    },
    add_item(item) {
      item = { ...item };
      if (item.has_variants) {
        evntBus.$emit("open_variants_model", item, this.items);
      } else {
        if (!item.qty || item.qty === 1) {
          item.qty = Math.abs(this.qty);
        }
        evntBus.$emit('add_item', item);
        // in case custom_item_add_on is set, then add the add-on item
        if (item.custom_item_add_on) {
          let index = this.items.findIndex(
            (el) =>
              el.item_code === item.custom_item_add_on
          );
          let add_on_item = this.items[index];
          add_on_item.qty = item.qty;
          evntBus.$emit('add_item', add_on_item);
          evntBus.$emit('show_mesage', {
            text: __(`added add-on item {0}`, [
              add_on_item.item_name,
            ]),
            color: 'success',
          });
        }
        this.qty = 1;
      }
    },
    click_add_item(item) {
      this.first_search = item.item_code;
      this.enter_qty();
    },

    verify_input_qty(new_item) {
      return new Promise((resolve, reject) => {
        // custom_uom_int is a custom field added in Item doctype (which pulls custom_uom_int from must_be_whole_number in UOM doctype) for verifying fractional inputs
        if (new_item.custom_uom_int == 1 && ((new_item.qty - parseInt(new_item.qty)) > 0.0000001)) {
          evntBus.$emit('show_mesage', {
            text: __(`QTY Must be Whole Number`),
            color: 'error',
          });
          frappe.utils.play_sound('error');
          resolve(false);
        } else resolve(true);
      })
    },

    async enter_event() {
      let match = false;
      if (!this.filtred_items.length || !this.first_search) {
        return;
      }
      const qty = this.get_item_qty(this.first_search);   // does Math.abs on qty, and checks for weight (qty) from barcodes printed by weighing scale..
      const new_item = { ...this.filtred_items[0] };
      new_item.qty = flt(qty);
      new_item.item_barcode.forEach((element) => {
        if (this.search == element.barcode) {
          new_item.uom = element.posa_uom;
          match = true;
        }
      });
      if (
        !new_item.to_set_serial_no &&
        new_item.has_serial_no &&
        this.pos_profile.posa_search_serial_no
      ) {
        new_item.serial_no_data.forEach((element) => {
          if (this.search && element.serial_no == this.search) {
            new_item.to_set_serial_no = this.first_search;
            match = true;
          }
        });
      }
      if (this.flags.serial_no) {
        new_item.to_set_serial_no = this.flags.serial_no;
      }
      if (
        !new_item.to_set_batch_no &&
        new_item.has_batch_no &&
        this.pos_profile.posa_search_batch_no
      ) {
        new_item.batch_no_data.forEach((element) => {
          if (this.search && element.batch_no == this.search) {
            new_item.to_set_batch_no = this.first_search;
            new_item.batch_no = this.first_search;
            match = true;
          }
        });
      }
      if (this.pos_profile.posa_regular_search)
        match = true; // enables Item-Code/Name based search (else only batch/serial based search words); this option is enabled from POS Profile.
      if (this.flags.batch_no) {
        new_item.to_set_batch_no = this.flags.batch_no;
      }
      if (match) {
        console.log("before 'if (await this.verify_input_qty(new_item))'");
        if (await this.verify_input_qty(new_item))
          this.add_item(new_item);
        this.search = null;
        this.first_search = null;
        this.debounce_search = null;
        this.flags.serial_no = null;
        this.flags.batch_no = null;
        this.qty = 1;
        this.$refs.debounce_search.focus();
      }
    },
    search_onchange() {
      const vm = this;
      if (vm.pos_profile.pose_use_limit_search) {
        vm.get_items();
      } else {
        //vm.enter_event();
        vm.enter_qty();
      }
    },
    enter_qty() {
      this.qty = '';
      this.$refs.input_qty.focus();
    },

    scale_button() {
      this.$refs.weighing_scale.$el.focus();
    },

    get_weight_from_scale() {
      let verify_scale_reading = true;
      const isDigit_isPeriod = (character) => {
        const code = character.codePointAt(0);
        return (code > 47 && code < 58) || code == 46;
      }

      if (this.scale_port_promise) {
        let weight_string_array = [];

        const qty_promise = this.scale_port_promise.then((port) => {
          return port.open({ baudRate: 9600 }).then(() => {   // Open connection to weighing scale
            /* evntBus.$emit('show_mesage', {
              text: `Weighing Scale Connected successfully`,
              color: 'success',
            }); */

            const textDecoder = new TextDecoderStream();
            const readableStreamClosed = port.readable.pipeTo(textDecoder.writable);
            const reader = textDecoder.readable.getReader();

            function reader_loop() {
              return reader.read().then(({ value, done }) => {
                if (done) {     // in case the port is closed from sender..
			            console.log('[readLoop] DONE', done);
                  // reader.releaseLock();
                  // return;
                  reader.cancel();
                  return readableStreamClosed.then(
                    error => {
                      /* Ignore the error */
                      console.log("Ignore the error: ", error);
                    },
                    () => {
                      return port.close().then(() => {
                        return parseFloat(weight_string_array.join(''));
                      })
                    }
                  );
		            }
		            if (value) {
                  // console.log("value = "+value);
                  if (!value.match(/^[\x00-\x7F]+$/g)) {
                    // checks for non-ASCII characters in the value
                    // we can also use this 'check' for assessing whether the BaudRate is correct for connected scale..
                    evntBus.$emit('show_mesage', {
                      text: `Receiving un-readable characters, kindly check the scale connectivity/settings, and try again`,
                      color: 'error',
                    });
                    console.log('Receiving un-readable characters, kindly check the scale connectivity/settings, and try again');
                    reader.cancel();
                    return readableStreamClosed.then(
                      error => {
                        /* Ignore the error */
                        console.log("Ignore the error: ", error);
                      },
                      () => {
                        return port.close().then(() => {
                          return;
                        })
                      }
                    );
                  }

                  for (let character of value) {
                    if (character.codePointAt(0) == 45) {   // Minus character
                      evntBus.$emit('show_mesage', {
                        text: `Receiving Minus (-) Weight Reading, kindly reset and weigh again`,
                        color: 'error',
                      });
                      console.log('Receiving Minus (-) Weight Reading, kindly reset and weigh again');
                      reader.cancel();
                      return readableStreamClosed.then(
                        error => {
                          /* Ignore the error */
                          console.log("Ignore the error: ", error);
                        },
                        () => {
                          return port.close().then(() => {
                            return;
                          })
                        }
                      );
                    }

                    if (isDigit_isPeriod(character)) {
                      weight_string_array.push(character);
                      // console.log("weight_string_array = "+weight_string_array);
                    }

                    if (weight_string_array.length == 7 && weight_string_array[3].codePointAt(0) == 46) {
                      // in case the array has 7 characters, with first 3 characters as digits, then decimal at 4th character,
                      // then 3 digits, then this is a valid weight reading - join, parseFloat & return this array, after closing the port

                      // console.log("final weight_string_array: " + weight_string_array);
                      // console.log("Final weight string after parseFloat: ", parseFloat(weight_string_array.join('')));
                      // console.log('[readLoop] DONE, calling reader.cancel()');

                      reader.cancel();
                      return readableStreamClosed.then(
                        error => {
                          /* Ignore the error */
                          console.log("Ignore the error: ", error);
                        },
                        () => {
                          return port.close().then(() => {
                            return parseFloat(weight_string_array.join(''));
                          })
                        }
                      );
                    } else if (weight_string_array.length > 7) {
                      /* evntBus.$emit('show_mesage', {
                        text: `Data Loss in Serial Port: taking the Weight Reading again`,
                        color: 'error',
                      }); */
                      weight_string_array = [];
                    }
                  }
			            return reader_loop();
		            }
	            })

            }
            return reader_loop();

          }).catch((error) => {
            // port not opened successfully
            return port.close().then(() => {
              return evntBus.$emit('show_mesage', {
                text: `Port.open block : "${error}"`,
                color: 'error',
              });
            })
          })

        })
        qty_promise.then((qty) => {
          this.qty = qty;
        })

      } else {
        verify_scale_reading = false;
        evntBus.$emit('show_mesage', {
          text: `Scale not connected - please press the 'ALLOW scale' button`,
          color: 'error',
        });
        evntBus.$emit('request_scale_port'); // pass event to Navbar.vue
      }
      if (verify_scale_reading) {
        this.$refs.input_qty.focus();;
      }
    },

    checkout() {
      evntBus.$emit('checkout');
    },
    get_item_qty(first_search) {
      let scal_qty = Math.abs(this.qty);
      if (first_search.startsWith(this.pos_profile.posa_scale_barcode_start)) {
        let pesokg1 = first_search.substr(7, 5);
        let pesokg;
        if (pesokg1.startsWith("0000")) {
          pesokg = "0.00" + pesokg1.substr(4);
        } else if (pesokg1.startsWith("000")) {
          pesokg = "0.0" + pesokg1.substr(3);
        } else if (pesokg1.startsWith("00")) {
          pesokg = "0." + pesokg1.substr(2);
        } else if (pesokg1.startsWith("0")) {
          pesokg =
            pesokg1.substr(1, 1) + "." + pesokg1.substr(2, pesokg1.length);
        } else if (!pesokg1.startsWith("0")) {
          pesokg =
            pesokg1.substr(0, 2) + "." + pesokg1.substr(2, pesokg1.length);
        }
        scal_qty = pesokg;
      }
      return scal_qty;
    },
    get_search(first_search) {
      let search_term = "";
      if (
        first_search &&
        first_search.startsWith(this.pos_profile.posa_scale_barcode_start)
      ) {
        search_term = first_search.substr(0, 7);
      } else {
        search_term = first_search;
      }
      return search_term;
    },
    esc_event() {
      this.search = null;
      this.first_search = null;
      this.qty = 1;
      this.$refs.debounce_search.focus();
    },
    update_items_details(items) {
      // set debugger
      const vm = this;
      frappe.call({
        method: "posawesome.posawesome.api.posapp.get_items_details",
        args: {
          pos_profile: vm.pos_profile,
          items_data: items,
        },
        callback: function (r) {
          if (r.message) {
            items.forEach((item) => {
              const updated_item = r.message.find(
                (element) => element.item_code == item.item_code
              );
              item.actual_qty = updated_item.actual_qty;
              item.serial_no_data = updated_item.serial_no_data;
              item.batch_no_data = updated_item.batch_no_data;
              item.item_uoms = updated_item.item_uoms;
            });
          }
        },
      });
    },
    update_cur_items_details() {
      this.update_items_details(this.filtred_items);
    },
    scan_barcoud() {
      const vm = this;
      onScan.attachTo(document, {
        suffixKeyCodes: [],
        keyCodeMapper: function (oEvent) {
          oEvent.stopImmediatePropagation();
          return onScan.decodeKeyEvent(oEvent);
        },
        onScan: function (sCode) {
          setTimeout(() => {
            vm.trigger_onscan(sCode);
          }, 300);
        },
      });
    },
    trigger_onscan(sCode) {
      if (this.filtred_items.length == 0) {
        evntBus.$emit("show_mesage", {
          text: `No Item has this barcode "${sCode}"`,
          color: "error",
        });
        frappe.utils.play_sound("error");
      } else {
        this.enter_qty();
        //this.enter_event();
        //this.debounce_search = null;
        //this.search = null;
      }
    },
    generateWordCombinations(inputString) {
      const words = inputString.split(" ");
      const wordCount = words.length;
      const combinations = [];

      // Helper function to generate all permutations
      function permute(arr, m = []) {
        if (arr.length === 0) {
          combinations.push(m.join(" "));
        } else {
          for (let i = 0; i < arr.length; i++) {
            const current = arr.slice();
            const next = current.splice(i, 1);
            permute(current.slice(), m.concat(next));
          }
        }
      }

      permute(words);

      return combinations;
    },
  },

  computed: {
    filtred_items() {
      this.search = this.get_search(this.first_search);
      if (!this.pos_profile.pose_use_limit_search) {
        let filtred_list = [];
        let filtred_group_list = [];
        if (this.item_group != "ALL") {
          filtred_group_list = this.items.filter((item) =>
            item.item_group
              .toLowerCase()
              .includes(this.item_group.toLowerCase())
          );
        } else {
          filtred_group_list = this.items;
        }
        // if (!this.search || this.search.length < 3) {
        if (!this.search) { // reduced minimum search count to 1, as PT also has 1 digit item codes
          if (
            this.pos_profile.posa_show_template_items &&
            this.pos_profile.posa_hide_variants_items
          ) {
            return (filtred_list = filtred_group_list
              .filter((item) => !item.variant_of)
              .slice(0, 50));
          } else {
            return (filtred_list = filtred_group_list.slice(0, 50));
          }
        } else if (this.search) {
          filtred_list = filtred_group_list.filter((item) => {
            let found = false;
            for (let element of item.batch_no_data) {
              if (element.batch_barcode == this.search || element.batch_no == this.search) { // Matching with Batch Barcode
                found = true;
                this.flags.batch_no = null;
                this.flags.batch_no = this.search;
                break;
              }
            }
            if (found)
              return found;
            else {
              for (let element of item.item_barcode) {
                if (element.barcode == this.search) { // Matching with Batch Barcode
                  found = true;
                  break;
                }
              }
            return found;
            }
          });
          /* filtred_list = filtred_group_list.filter((item) => {
            let found = false;
            for (let element of item.item_barcode) {
              if (element.barcode == this.search) {
                found = true;
                break;
              }
            }
            return found;
          }); */
          if (filtred_list.length == 0) {
            filtred_list = filtred_group_list.filter((item) => {
              //item.item_code.toLowerCase().includes(this.search.toLowerCase())
              return item.item_code == this.search;     // does an exact match of item codes
            }
            );
            if (filtred_list.length == 0) {
              // search for specific words in the 'Item Names'
              filtred_list = filtred_group_list.filter((item) => {
                searchText = item.item_name.toLowerCase();
                queryText = this.search.toLowerCase();
                search_index = searchText.indexOf(queryText);
                return search_index > -1;
              })
            };
            if (filtred_list.length == 0) {
              const search_combinations = this.generateWordCombinations(
                this.search
              );
              filtred_list = filtred_group_list.filter((item) => {
                let found = false;
                for (let element of search_combinations) {
                  element = element.toLowerCase().trim();
                  let element_regex = new RegExp(
                    `.*${element.split("").join(".*")}.*`
                  );
                  if (element_regex.test(item.item_name.toLowerCase())) {
                    found = true;
                    break;
                  }
                }
                return found;
              });
            }
            if (
              filtred_list.length == 0 &&
              this.pos_profile.posa_search_serial_no
            ) {
              filtred_list = filtred_group_list.filter((item) => {
                let found = false;
                for (let element of item.serial_no_data) {
                  if (element.serial_no == this.search) {
                    found = true;
                    this.flags.serial_no = null;
                    this.flags.serial_no = this.search;
                    break;
                  }
                }
                return found;
              });
            }
            /* if (
              filtred_list.length == 0 &&
              this.pos_profile.posa_search_batch_no
            ) {
              filtred_list = filtred_group_list.filter((item) => {
                let found = false;
                for (let element of item.batch_no_data) {
                  if (element.batch_barcode == this.search || element.batch_no == this.search) { // Matching with Batch Barcode
                    found = true;
                    this.flags.batch_no = null;
                    this.flags.batch_no = this.search;
                    break;
                  }
                }
                return found;
              });
            } */
          }
        }
        if (
          this.pos_profile.posa_show_template_items &&
          this.pos_profile.posa_hide_variants_items
        ) {
          return filtred_list.filter((item) => !item.variant_of).slice(0, 50);
        } else {
          return filtred_list.slice(0, 50);
        }
      } else {
        return this.items.slice(0, 50);
      }
    },
    debounce_search: {
      get() {
        return this.first_search;
      },
      set: _.debounce(function (newValue) {
        this.first_search = newValue;
      }, 200),
    },
  },

  created: function () {
    this.$nextTick(function () {});
    evntBus.$on("register_pos_profile", (data) => {
      this.pos_profile = data.pos_profile;
      this.get_items();
      this.get_items_groups();
      this.items_view = this.pos_profile.posa_default_card_view
        ? "card"
        : "list";
    });
    evntBus.$on('select_items', () => {     // $emit from Customer.vue
      this.$refs.debounce_search.focus();
    });
    evntBus.$on('scale_port_promise', (scale_port_promise) => {     // $emit (receive the scale_port_promise object) from invoice.vue
      this.scale_port_promise = scale_port_promise;
    });
    evntBus.$on('select_items', () => {     // $emit from Customer.vue
      this.$refs.debounce_search.focus();
    });
    evntBus.$on('scale_port_promise', (scale_port_promise) => {     // $emit (receive the scale_port_promise object) from invoice.vue
      this.scale_port_promise = scale_port_promise;
    });
    evntBus.$on("update_cur_items_details", () => {
      this.update_cur_items_details();
    });
    evntBus.$on("update_offers_counters", (data) => {
      this.offersCount = data.offersCount;
      this.appliedOffersCount = data.appliedOffersCount;
    });
    evntBus.$on("update_coupons_counters", (data) => {
      this.couponsCount = data.couponsCount;
      this.appliedCouponsCount = data.appliedCouponsCount;
    });
    evntBus.$on("update_customer_price_list", (data) => {
      this.customer_price_list = data;
    });
    evntBus.$on("update_customer", (data) => {
      this.customer = data;
    });
  },

  mounted() {
    this.scan_barcoud();
  },
};
</script>

<style scoped></style>
