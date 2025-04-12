<template>
  <div class="customer-search-container">
    <v-autocomplete
      density="comfortable"
      variant="outlined"
      color="deep-purple-darken-2"
      bg-color="grey-lighten-4"
      :label="__('Customer')"
      v-model="customer"
      :items="filteredCustomers"
      item-title="customer_name"
      item-value="name"
      :disabled="readonly"
      append-icon="mdi-plus-circle-outline"
      @click:append="new_customer"
      prepend-inner-icon="mdi-account-edit-outline"
      @click:prepend-inner="edit_customer"
      return-object
      @update:model-value="onCustomerSelect"
      clearable
      clear-icon="mdi-close-circle-outline"
      :loading="loading"
      v-model:search="searchInput"
      :no-filter="true"
      class="customer-autocomplete"
      :menu-props="{ contentClass: 'customer-menu' }"
    >
      <template v-slot:item="{ props, item }">
        <v-list-item
          v-bind="props"
          :title="item.raw.customer_name || item.raw.name"
          :subtitle="getCustomerSubtitle(item.raw)"
          class="customer-list-item"
        >
          <template v-slot:title>
            <span class="customer-name">{{ item.raw.customer_name || item.raw.name }}</span>
          </template>
          <template v-slot:subtitle>
            <div class="customer-details">
              <div v-if="item.raw.name" class="detail-item id-badge">
                <v-icon small class="mr-1">mdi-identifier</v-icon>
                {{ item.raw.name }}
              </div>
              <div v-if="item.raw.tax_id" class="detail-item tax-badge">
                <v-icon small class="mr-1">mdi-card-account-details-outline</v-icon>
                {{ item.raw.tax_id }}
              </div>
              <div v-if="item.raw.email_id" class="detail-item email-badge">
                <v-icon small class="mr-1">mdi-email-outline</v-icon>
                {{ item.raw.email_id }}
              </div>
              <div v-if="item.raw.mobile_no" class="detail-item phone-badge">
                <v-icon small class="mr-1">mdi-phone-outline</v-icon>
                {{ item.raw.mobile_no }}
              </div>
              <div v-if="item.raw.primary_address" class="detail-item address-badge">
                <v-icon small class="mr-1">mdi-map-marker-outline</v-icon>
                {{ formatAddress(item.raw) }}
              </div>
            </div>
          </template>
        </v-list-item>
      </template>
      
      <template v-slot:no-data>
        <v-list-item class="no-results-item">
          <v-list-item-title>
            <v-icon color="grey-darken-1" class="mr-2">mdi-magnify-remove</v-icon>
            No customers matching "{{ searchInput }}"
          </v-list-item-title>
        </v-list-item>
      </template>
      
      <template v-slot:progress>
        <v-progress-linear
          v-if="loading"
          absolute
          color="deep-purple-accent-2"
          indeterminate
          height="3"
        ></v-progress-linear>
      </template>
      
      <template v-slot:append-inner>
        <v-icon
          v-if="customer"
          color="deep-purple-darken-2"
          @click.stop="customer = null"
          class="clear-btn"
        >
          mdi-close-circle
        </v-icon>
      </template>
    </v-autocomplete>
    
    <div class="update-customer-container">
      <UpdateCustomer />
    </div>
  </div>
</template>

<script>
import { evntBus } from '../../bus';
import UpdateCustomer from './UpdateCustomer.vue';

export default {
  data: () => ({
    pos_profile: '',
    customers: [],
    customer: null,
    readonly: false,
    customer_info: {},
    loading: false,
    searchInput: '',
  }),

  components: {
    UpdateCustomer,
  },

  computed: {
    filteredCustomers() {
      if (!this.searchInput) return this.customers;
      
      const searchText = this.searchInput.toLowerCase().trim();
      
      return this.customers.filter(customer => {
        const searchFields = [
          customer.customer_name?.toLowerCase() || '',
          customer.name?.toLowerCase() || '',
          customer.name?.replace('CUST-', '').toLowerCase() || '',
          customer.name?.split('-').pop().toLowerCase() || '',
          customer.tax_id?.toLowerCase() || '',
          customer.mobile_no?.toLowerCase() || '',
          this.formatAddress(customer)?.toLowerCase() || ''
        ];
        
        return searchFields.some(field => field.includes(searchText));
      });
    }
  },

  methods: {
    async get_customer_names() {
      const vm = this;
      if (this.customers.length > 0 && !this.searchInput) return;
      
      this.loading = true;
      
      try {
        // Try loading from cache first
        if (vm.pos_profile.posa_local_storage && localStorage.customer_storage) {
          try {
            vm.customers = JSON.parse(localStorage.getItem('customer_storage'));
          } catch (e) {
            console.error('Error parsing cached customers:', e);
          }
        }
        
        // Fetch from server
        await new Promise((resolve) => {
          frappe.call({
            method: 'posawesome.posawesome.api.posapp.get_customer_names',
            args: { pos_profile: this.pos_profile.pos_profile },
            callback: (r) => {
              if (r.message) {
                vm.customers = r.message;
                if (vm.pos_profile.posa_local_storage) {
                  try {
                    localStorage.setItem('customer_storage', JSON.stringify(r.message));
                  } catch (e) {
                    console.error('Error caching customers:', e);
                  }
                }
              }
              resolve();
            },
            error: (err) => {
              console.error('Error loading customers:', err);
              resolve();
            }
          });
        });
      } finally {
        this.loading = false;
      }
    },

    new_customer() {
      evntBus.emit('open_update_customer', null);
    },
    
    edit_customer() {
      if (this.customer) {
        evntBus.emit('open_update_customer', this.customer);
      }
    },
    
    getCustomerSubtitle(customer) {
      return [
        customer.name && `ID: ${customer.name}`,
        customer.tax_id && `TAX ID: ${customer.tax_id}`,
        customer.email_id && `Email: ${customer.email_id}`,
        customer.mobile_no && `Mobile No: ${customer.mobile_no}`,
        customer.primary_address && `Address: ${this.formatAddress(customer)}`
      ].filter(Boolean).join('\n');
    },
    
    formatAddress(customer) {
      if (!customer.primary_address) return '';
      return [
        customer.primary_address,
        customer.city,
        customer.state,
        customer.pincode,
        customer.country
      ].filter(Boolean).join(', ');
    },
    
    onCustomerSelect(customer) {
      if (!customer) return;
      this.customer_info = customer;
      evntBus.emit('update_customer', customer.name);
    }
  },

  created() {
    const setupEventListeners = () => {
      evntBus.on('register_pos_profile', (pos_profile) => {
        this.pos_profile = pos_profile;
        this.get_customer_names();
      });
      
      // Other event listeners...
      evntBus.on('set_customer', (customer) => {
        this.customer = this.customers.find(c => c.name === customer) || customer;
      });
      
      // Add all other event listeners similarly
    };
    
    this.$nextTick(setupEventListeners);
  },

  beforeUnmount() {
    // Clean up all event listeners
    evntBus.off('register_pos_profile');
    evntBus.off('payments_register_pos_profile');
    evntBus.off('set_customer');
    evntBus.off('add_customer_to_list');
    evntBus.off('set_customer_readonly');
    evntBus.off('set_customer_info_to_edit');
    evntBus.off('fetch_customer_details');
  }
};
</script>

<style scoped>
.customer-search-container {
  border-radius: 8px;
  padding: 8px;
  background: linear-gradient(to bottom, #f5f5f5, #eeeeee);
}

.customer-autocomplete {
  border-radius: 8px;
  box-shadow: 0 2px 8px rgba(0,0,0,0.1);
}

.customer-autocomplete :deep(.v-field__outline) {
  color: #7e57c2;
}

.customer-autocomplete :deep(.v-field--focused .v-field__outline) {
  color: #5e35b1;
}

.customer-list-item {
  border-bottom: 1px solid #e0e0e0;
  transition: all 0.3s ease;
}

.customer-list-item:hover {
  background-color: #f3e5f5;
}

.customer-name {
  font-weight: 600;
  color: #5e35b1;
  font-size: 1rem;
}

.customer-details {
  display: flex;
  flex-direction: column;
  gap: 6px;
  margin-top: 6px;
}

.detail-item {
  display: flex;
  align-items: center;
  font-size: 0.8rem;
  line-height: 1.3;
  padding: 4px 8px;
  border-radius: 4px;
  background-color: rgba(238, 238, 238, 0.5);
}

.id-badge {
  color: #4527a0;
  background-color: #ede7f6;
}

.tax-badge {
  color: #2e7d32;
  background-color: #e8f5e9;
}

.email-badge {
  color: #1565c0;
  background-color: #e3f2fd;
}

.phone-badge {
  color: #6a1b9a;
  background-color: #f3e5f5;
}

.address-badge {
  color: #e65100;
  background-color: #fff3e0;
}

.no-results-item {
  min-height: 48px;
  display: flex;
  align-items: center;
  justify-content: center;
  color: #616161;
}

.update-customer-container {
  margin-top: 16px;
  padding: 12px;
  background-color: white;
  border-radius: 8px;
  box-shadow: 0 1px 3px rgba(0,0,0,0.1);
}

.clear-btn {
  cursor: pointer;
  transition: all 0.2s ease;
}

.clear-btn:hover {
  transform: scale(1.1);
  color: #d32f2f !important;
}
</style>

<style>
/* Global styles for the menu */
.customer-menu {
  border-radius: 8px !important;
  box-shadow: 0 4px 12px rgba(0,0,0,0.15) !important;
  border: 1px solid #e0e0e0 !important;
}

.customer-menu .v-list-item {
  padding-left: 12px !important;
  padding-right: 12px !important;
}
</style>