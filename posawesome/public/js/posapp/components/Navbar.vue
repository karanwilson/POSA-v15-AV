<template>
  <nav>
    <v-app-bar app height="40" class="elevation-2">
      <v-app-bar-nav-icon
        @click.stop="drawer = !drawer"
        class="grey--text"
      ></v-app-bar-nav-icon>
      <v-img
        src="/assets/posawesome/js/posapp/components/pos/pos.png"
        alt="POS Awesome"
        max-width="32"
        class="mr-2"
        color="primary"
      ></v-img>
      <v-toolbar-title
        @click="go_desk"
        style="cursor: pointer"
        class="text-uppercase primary--text"
      >
        <span class="font-weight-light">pos</span>
        <span>awesome</span>
      </v-toolbar-title>

      <v-spacer></v-spacer>

      <v-col
        v-if="pos_profile.posa_enable_fs_payments"
        cols="1"
        align="center"
      >
        <v-switch
          v-model="fs_online"
          :label="frappe._('FS Online')"
          hide-details
        ></v-switch>
      </v-col>

      <v-col
        v-if="pos_profile.posa_enable_fs_payments"
        cols="1"
        align="center"
      >
        <v-btn
          icon
          text
          :color="dynamic_fs_online_color"
        >
          <v-icon>{{ dynamic_fs_online_icon }}</v-icon>
          fs login
        </v-btn>
      </v-col>

      <v-col
        v-if="pos_profile.posa_input_qty && pos_profile.posa_input_weighing_scale"
        cols="1"
        align="center"
      >
        <v-btn
          text
          icon
          :color="dynamic_scale_color"
          @click="request_scale_port"
          ref="allow_scale_button"
        >
          access
          <v-icon>mdi-scale</v-icon>
        </v-btn>
      </v-col>

      <v-btn style="cursor: unset" text color="primary">
        <span right>{{ pos_profile.name }}</span>
      </v-btn>
      <div class="text-center">
        <v-menu offset-y>
          <template v-slot:activator="{ on, attrs }">
            <v-btn color="primary" dark text v-bind="attrs" v-on="on"
              >Menu</v-btn
            >
          </template>
          <v-card class="mx-auto" max-width="300" tile>
            <v-list dense>
              <v-list-item-group v-model="menu_item" color="primary">
                <v-list-item
                  @click="close_shift_dialog"
                  v-if="!pos_profile.posa_hide_closing_shift && item == 0"
                >
                  <v-list-item-icon>
                    <v-icon>mdi-content-save-move-outline</v-icon>
                  </v-list-item-icon>
                  <v-list-item-content>
                    <v-list-item-title>{{
                      __('Close Shift')
                    }}</v-list-item-title>
                  </v-list-item-content>
                </v-list-item>
                <v-list-item
                  @click="print_last_invoice"
                  v-if="
                    pos_profile.posa_allow_print_last_invoice &&
                    this.last_invoice
                  "
                >
                  <v-list-item-icon>
                    <v-icon>mdi-printer</v-icon>
                  </v-list-item-icon>
                  <v-list-item-content>
                    <v-list-item-title>{{
                      __('Print Last Invoice')
                    }}</v-list-item-title>
                  </v-list-item-content>
                </v-list-item>
                <v-divider class="my-0"></v-divider>
                <v-list-item @click="logOut">
                  <v-list-item-icon>
                    <v-icon>mdi-logout</v-icon>
                  </v-list-item-icon>
                  <v-list-item-content>
                    <v-list-item-title>{{ __('Logout') }}</v-list-item-title>
                  </v-list-item-content>
                </v-list-item>
                <v-list-item @click="go_about">
                  <v-list-item-icon>
                    <v-icon>mdi-information-outline</v-icon>
                  </v-list-item-icon>
                  <v-list-item-content>
                    <v-list-item-title>{{ __('About') }}</v-list-item-title>
                  </v-list-item-content>
                </v-list-item>
              </v-list-item-group>
            </v-list>
          </v-card>
        </v-menu>
      </div>
    </v-app-bar>
    <v-navigation-drawer
      v-model="drawer"
      :mini-variant.sync="mini"
      app
      class="primary margen-top"
      width="170"
    >
      <v-list dark>
        <v-list-item class="px-2">
          <v-list-item-avatar>
            <v-img :src="company_img"></v-img>
          </v-list-item-avatar>

          <v-list-item-title>{{ company }}</v-list-item-title>

          <v-btn icon @click.stop="mini = !mini">
            <v-icon>mdi-chevron-left</v-icon>
          </v-btn>
        </v-list-item>
        <!-- <MyPopup/> -->
        <v-list-item-group v-model="item" color="white">
          <v-list-item
            v-for="item in items"
            :key="item.text"
            @click="changePage(item.text)"
          >
            <v-list-item-icon>
              <v-icon v-text="item.icon"></v-icon>
            </v-list-item-icon>
            <v-list-item-content>
              <v-list-item-title v-text="item.text"></v-list-item-title>
            </v-list-item-content>
          </v-list-item>
        </v-list-item-group>
      </v-list>
    </v-navigation-drawer>
    <v-snackbar v-model="snack" :timeout="5000" :color="snackColor" top right>
      {{ snackText }}
    </v-snackbar>
    <v-dialog v-model="freeze" persistent max-width="290">
      <v-card>
        <v-card-title class="text-h5">
          {{ freezeTitle }}
        </v-card-title>
        <v-card-text>{{ freezeMsg }}</v-card-text>
      </v-card>
    </v-dialog>
  </nav>
</template>

<script>
import { evntBus } from '../bus';

export default {
  // components: {MyPopup},
  data() {
    return {
      drawer: false,
      mini: true,
      item: 0,
      items: [{ text: 'POS', icon: 'mdi-network-pos' }],
      page: '',
      fav: true,
      menu: false,
      message: false,
      hints: true,
      menu_item: 0,
      snack: false,
      snackColor: '',
      snackText: '',
      company: 'POS Awesome',
      company_img: '/assets/erpnext/images/erpnext-logo.svg',
      pos_profile: '',
      freeze: false,
      freezeTitle: '',
      freezeMsg: '',
      last_invoice: '',
      fs_online: false, // for checking whether FS server is online or offline
      dynamic_scale_color: 'grey-darken-4', // for dynamically setting color based on browser compatibility
      dynamic_fs_online_color: 'error', // 'success'
      dynamic_fs_online_icon: 'mdi-server-network-off', // 'mdi-server-network'
    };
  },
  methods: {
    fapi_login() {
      const vm = this;
      frappe.call({
        method: 'payments.payment_gateways.doctype.fs_settings.fs_settings.login',
        callback: function (r) {
          if (r.message = 'OK') {
            vm.dynamic_fs_online_color = 'success';
            vm.dynamic_fs_online_icon = 'mdi-server-network';
          }
        },
      });
    },
    fapi_logout() {
      const vm = this;
      frappe.call({
        method: 'payments.payment_gateways.doctype.fs_settings.fs_settings.logout',
        callback: function (r) {
          if (r.message = 'OK') {
            vm.dynamic_fs_online_color = 'error';
            vm.dynamic_fs_online_icon = 'mdi-server-network-off';
          }
        },
      });
    },
    // Request Serial Port for weighing Scale
    request_scale_port() {
      if ("serial" in navigator) {
        this.dynamic_scale_color = 'success';
        evntBus.$emit('show_mesage', {
          text: `Weighing Scale Connectivity supported`,
          color: 'success',
        });

        const scale_port_promise = navigator.serial.getPorts().then((ports) => {
          let filters = [];
          if (ports) {
            for (let port of ports) {
              const { usbProductId, usbVendorId } = port.getInfo();
              if (usbProductId & usbVendorId) {
                filters.push({ usbProductId, usbVendorId });
              }
            }
            // console.log("known ports: "+filters);
          }

          return navigator.serial.requestPort({ filters }).then((port) => {
            return port;
          }).catch((error) => {
            // The user didn't select a port.
            evntBus.$emit('show_mesage', {
              text: error,
              color: 'error',
            });
          });
        })

        // console.log(scale_port_promise);
        evntBus.$emit('scale_port_promise', scale_port_promise);    // pass event to ItemsSelector.vue
        evntBus.$emit('input_customer');    // pass event to Customer.vue

      } else {
        // browser not supported by web-serial-api
        this.dynamic_scale_color = 'error';
        evntBus.$emit('show_mesage', {
          text: `Weighing Scale Connects only with Chrome, Edge, Vivaldi, Opera, or any Chromium browser`,
          color: 'error',
        });
        evntBus.$emit('input_customer');    // pass event to Customer.vue
      }
    },

    changePage(key) {
      this.$emit('changePage', key);
    },
    go_desk() {
      frappe.set_route('/');
      location.reload();
    },
    go_about() {
      const win = window.open(
        'https://github.com/yrestom/POS-Awesome',
        '_blank'
      );
      win.focus();
    },
    close_shift_dialog() {
      evntBus.$emit('open_closing_dialog');
    },
    show_mesage(data) {
      this.snack = true;
      this.snackColor = data.color;
      this.snackText = data.text;
    },
    logOut() {
      var me = this;
      me.logged_out = true;
      return frappe.call({
        method: 'logout',
        callback: function (r) {
          if (r.exc) {
            return;
          }
          frappe.set_route('/login');
          location.reload();
        },
      });
    },
    print_last_invoice() {
      if (!this.last_invoice) return;
      const print_format =
        this.pos_profile.print_format_for_online ||
        this.pos_profile.print_format;
      const letter_head = this.pos_profile.letter_head || 0;
      const url =
        frappe.urllib.get_base_url() +
        '/printview?doctype=Sales%20Invoice&name=' +
        this.last_invoice +
        '&trigger_print=1' +
        '&format=' +
        print_format +
        '&no_letterhead=' +
        letter_head;
      const printWindow = window.open(url, 'Print');
      printWindow.addEventListener(
        'load',
        function () {
          printWindow.print();
        },
        true
      );
    },
  },
  created: function () {
    this.$nextTick(function () {
      evntBus.$on('show_mesage', (data) => {
        this.show_mesage(data);
      });
      evntBus.$on('set_company', (data) => {
        this.company = data.name;
        this.company_img = data.company_logo
          ? data.company_logo
          : this.company_img;
      });
      evntBus.$on('register_pos_profile', (data) => {
        this.pos_profile = data.pos_profile;
        const payments = { text: 'Payments', icon: 'mdi-cash-register' };
        if (
          this.pos_profile.posa_use_pos_awesome_payments &&
          this.items.length !== 2
        ) {
          this.items.push(payments);
        }
        this.$nextTick(function() {     // to wait for $el to be initialised
          this.fs_online = true;
          if (this.pos_profile.posa_input_qty && this.pos_profile.posa_input_weighing_scale) {
            this.$refs.allow_scale_button.$el.focus();   // request permission for accessing the scale port
            console.info('request_scale_port');
          }
        });
      });
      evntBus.$on('request_scale_port', () => {       // $emit from ItemsSelector.vue
        this.$refs.allow_scale_button.$el.focus();   // request permission for accessing the scale port
      });
      evntBus.$on('set_last_invoice', (data) => {
        this.last_invoice = data;
      });
      evntBus.$on('freeze', (data) => {
        this.freeze = true;
        this.freezeTitle = data.title;
        this.freezeMsg = data.msg;
      });
      evntBus.$on('unfreeze', () => {
        this.freeze = false;
        this.freezTitle = '';
        this.freezeMsg = '';
      });
    });
  },
  watch: {
    fs_online(value) {
      if (value == 1) this.fapi_login()
      //else this.fapi_logout() // logout function is not working/defined in the FS sandbox API
    }
  }
};
</script>

<style scoped>
.margen-top {
  margin-top: 0px;
}
</style>
