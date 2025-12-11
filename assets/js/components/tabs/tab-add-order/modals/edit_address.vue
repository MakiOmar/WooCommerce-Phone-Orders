<template>
    <div>
        <b-modal id="editAddress"
                 ref="modal"
                 :title="editAddressLabel"
                 @shown="formLoad"
                 @hidden="hideModal"
                 size="lg"
		 :noCloseOnBackdrop="modalDontCloseOnBackdropClick"
		 :static="true"
         v-model="showModal"
        >
            <b-container>

		<slot
		    name="multi-addresses-select-slot"
		    :customer-group-fields="groupFields"
		    :customer="customer"
		    :address-type="addressType"
		></slot>

		<slot name="edit-customer-address-modal-header" modal="editAddressModal"></slot>

                <b-row v-for="(group, groupKey) in groupFields" :key="groupKey">

                    <b-col cols="12">

                        <google-autocomplete
                            ref="google_autocomplete"
                            :input-placeholder="autocompleteInputPlaceholder"
                            :invalid-message="autocompleteInvalidMessage"
			    :custom-google-autocomplete-js-callback="customGoogleAutocompleteJsCallback"
                            @change="updateAddress"
                            v-if="groupKey === 'address' && !!!initCustomAutocompleteFunction"
                            v-show="!!googleAutocompleteAPIKey && !!group.rows.length"
                        ></google-autocomplete>

			<custom-autocomplete
			    ref="custom_autocomplete"
			    :input-placeholder="initCustomAutocompletePlaceholder"
			    :init-autocomplete-function="initCustomAutocompleteFunction"
			    @change="updateAddress"
			    v-if="groupKey === 'address' && !!initCustomAutocompleteFunction"
			    v-show="!!group.fields.length"
			></custom-autocomplete>

                        <b-row v-for="(row, index) in group.rows" :key="index">

                            <b-col v-for="(loopField, numIndex) in row" :md="countCols" cols="12" :key="loopField.key" v-show="isFieldVisible(loopField.key)">

				<span>

                                    <strong><label class="mr-sm-2" :class="[ emptyRequiredFieldsList.indexOf(loopField.key) > -1 ? 'text-danger' : '' ]">{{ requiredFieldsList.indexOf(loopField.key) > -1 ? loopField.label + ' *' : loopField.label }}</label></strong>

				    <div v-if="loopField.key === 'country'">
					<multiselect
					    :ref="loopField.key"
					    :allow-empty="false"
					    :hide-selected="true"
					    :searchable="true"
					    label="title"
					    v-model="loopField.value"
					    :options="defaultCountryList"
					    track-by="value"
					    @update:model-value="onChangeDefaultCountry(loopField.key)"
					    :show-labels="false"
					    :placeholder="selectPlaceholder"
					>
                        <template v-slot:singleLabel="props">
                            <span>
                                <span v-html="props.option.title"></span>
                            </span>
                        </template>
                        <template v-slot:option="props">
                            <span>
                                <span v-html="props.option.title"></span>
                            </span>
                        </template>
                        <template v-slot:noOptions>
                            <span>
                                <span v-html="noOptionsTitle"></span>
                            </span>
                        </template>
					</multiselect>
				    </div>

				    <div v-else-if="loopField.key === 'state'">
					<multiselect
					    :ref="loopField.key"
					    v-if="statesList.length"
					    :allow-empty="false"
					    :hide-selected="true"
					    :searchable="true"
					    label="title"
					    v-model="loopField.value"
					    :options="statesList"
					    track-by="value"
					    @update:model-value="onEnter(loopField.key)"
					    :show-labels="false"
					    :placeholder="selectPlaceholder"
					>
					    <template v-slot:singleLabel="props">
                            <span>
                                <span v-html="props.option.title"></span>
                            </span>
                        </template>
                        <template v-slot:option="props">
                            <span>
                                <span v-html="props.option.title"></span>
                            </span>
                        </template>
                        <template v-slot:noOptions>
                            <span>
                                <span v-html="noOptionsTitle"></span>
                            </span>
                        </template>
					</multiselect>
					<b-form-input
					    v-else
					    :ref="loopField.key"
					    type="text"
					    class="mb-2 mr-sm-2 mb-sm-0"
					    v-model="loopField.value"
					    @keydown.enter="onEnter(loopField.key)"
					>
					</b-form-input>
				    </div>

				    <div v-else-if="loopField.key === 'role'">
					<multiselect
					    :ref="loopField.key"
					    :allow-empty="false"
					    v-model="loopField.value"
					    :options="rolesList"
					    @update:model-value="onEnter(loopField.key)"
					    :placeholder="selectPlaceholder"
					    :show-labels="false"
					    label="title"
					    track-by="value"
					>
                        <template v-slot:noOptions>
                            <span>
                                <span v-html="noOptionsTitle"></span>
                            </span>
                        </template>
					</multiselect>
				    </div>

				    <div v-else-if="loopField.key === 'locale'">
					<multiselect
					    :ref="loopField.key"
					    :allow-empty="false"
					    v-model="loopField.value"
					    :options="languagesList"
					    @update:model-value="onEnter(loopField.key)"
					    :placeholder="selectPlaceholder"
					    :show-labels="false"
					    label="title"
					    track-by="value"
					>
					    <template v-slot:noOptions>
                            <span>
                                <span v-html="noOptionsTitle"></span>
                            </span>
                        </template>
					</multiselect>
				    </div>

                <div v-else-if="loopField.key === 'billing_company_checkbox'">
                    <b-form-checkbox
                        :ref="loopField.key"
                        v-model="loopField.value"
                        @input="onCompanyCheckboxChange"
                        value="1"
                        unchecked-value=""
                        class="mt-2"
                    >
                    </b-form-checkbox>
                </div>

				    <div v-else>
					<b-form-input
					    :ref="loopField.key"
					    type="text"
					    class="mb-2 mr-sm-2 mb-sm-0"
					    :class="{ 'is-invalid': companyValidationErrors[loopField.key] && companyValidationErrors[loopField.key].length > 0 }"
					    v-model="loopField.value"
					    @keydown.enter="onEnter(loopField.key)"
					    @input="clearCompanyFieldError(loopField.key)"
                                            :readonly="readonlyFieldsList.indexOf(loopField.key) > -1"
					>
					</b-form-input>
					<div v-if="companyValidationErrors[loopField.key] && companyValidationErrors[loopField.key].length > 0" class="invalid-feedback d-block">
					    <div v-for="(error, index) in companyValidationErrors[loopField.key]" :key="index">{{ error }}</div>
					</div>
				    </div>
				</span>
                            </b-col>
                        </b-row>
                    </b-col>
                </b-row>
            </b-container>
            <slot name="edit-customer-address-modal-footer" modal="editAddressModal"></slot>
            <template v-slot:footer>
            <div style="width: 100%">
		<b-row>
		    <b-col cols="12" md="3">
			<b-button class="wpo-copy-billing-address-btn" @click="copyFromBillingAddress" v-if="this.addressType === 'shipping'">{{ copyFromBillingAddressLabel }}</b-button>
		    </b-col>
		    <b-col cols="12" md="9" style="text-align: right">
			<b-alert :show="!!error" variant="danger" class="error-alert">{{ this.error }}</b-alert>
			<b-button @click="cancel()">{{ cancelLabel }}</b-button>
			<b-button @click="saveAddress()" variant="primary">{{ saveAddressLabel }}</b-button>
			<slot
			    name="multi-addresses-buttons-slot"
			    :customer-group-fields="groupFields"
			    :customer="customer"
			    :address-type="addressType"
			></slot>
		    </b-col>
		</b-row>
            </div>
            </template>
        </b-modal>
    </div>
</template>

<style>
    @media (max-width:767px) {
	.wpo-copy-billing-address-btn {
	    margin-bottom: 10px;
	}
    }
</style>

<script>

        import Multiselect from 'vue-multiselect';
        import GoogleAutocomplete from '../google-autocomplete.vue';
        import CustomAutocomplete from '../custom-autocomplete.vue';

	export default {
		props: {
                    cancelLabel: {
                        default: function () {
                            return 'Cancel';
                        }
                    },
                    editAddressLabel: {
                        default: function () {
                            return 'Edit address';
                        }
                    },
                    saveAddressLabel: {
                        default: function () {
                            return 'Done';
                        }
                    },
                    tabName: {
                        default: function () {
                            return 'add-order';
                        },
                    },
                    selectPlaceholder: {
                        default: function () {
                            return 'Select option';
                        },
                    },
                    autocompleteInputPlaceholder: {
                        default: function () {
                            return null;
                        },
                    },
                    autocompleteInvalidMessage: {
                        default: function () {
                            return null;
                        },
                    },
                    noOptionsTitle: {
                        default: function() {
                            return 'List is empty.';
                        }
                    },
                    personalFieldsOrder: {
                        default: function() {
                            return ['email', 'role', 'first_name', 'last_name', 'company', 'phone', 'locale'];
                        }
                    },
                    addressFieldsOrder: {
                        default: function() {
                            return ['country', 'address_1', 'billing_company_checkbox', 'address_2', 'city', 'state', 'postcode', 'billing_company_vat', 'short_address', 'building_number', 'address_second', 'district', 'postal_code'];
                        }
                    },
                    countFieldsInRow: {
                        default: function() {
                            return 2;
                        }
                    },
		    initCustomAutocompleteFunction: {
			default: function() {
			    return null;
			}
		    },
		    initCustomAutocompletePlaceholder: {
			default: function() {
			    return null;
			}
		    },
		    fillAllFieldsLabel: {
			default: function() {
			    return 'Please fill out all required fields!';
			}
		    },
		    customerAddressAdditionalKeys: {
			default: function() {
			    return {};
			}
		    },
		    customGoogleAutocompleteJsCallback: {
			default: function() {
			    return '';
			}
		    },
		    copyFromBillingAddressLabel: {
			default: function() {
			    return 'Copy from billing address';
			}
		    },
		    rolesList: {
			default: function () {
			    return [];
			},
	            },
		    languagesList: {
			default: function () {
			    return [];
			},
	            },
		    readonlyFields: {
			default: function () {
			    return [];
			},
	            },
		},
		created: function () {
                    this.$root.bus.$on('edit-customer-address', (data) => {
                        this.addressType = data.addressType;
                        this.customer = data.customer;
	                    this.customFields = {};
	                    this.customFieldsAtTop = {};
                        this.fieldsToShow = data.fields;
                        this.showModal = true;
                    });

		    this.$root.bus.$on('edit-customer-address-custom-fields-updated', (fields) => {
			this.customFields = fields;
		    });

		    this.$root.bus.$on('edit-customer-address-custom-fields-updated-at-top', (fields) => {
			this.customFieldsAtTop = fields;
		    });

                    this.$root.bus.$on('update-customer-request', (data) => {
                        this.updateCustomerRequest(data.customer, data.callback, data.params);
                    });

                    this.$root.bus.$on('edit-customer-update-address', (address) => {
                        this.updateAddress(address);
                    });
                },
		data: function () {
                    let $react_fields = this.initForm();
                    $react_fields['addressType'] = '';
                    $react_fields['customer'] = {};
                    $react_fields['fieldsToShow'] = {};
                    $react_fields['customFields'] = {};
                    $react_fields['customFieldsAtTop'] = {};
                    $react_fields['error']          = '';
                    $react_fields['error_code']	    = '';
                    $react_fields['emptyRequiredFieldsList'] = [];
                    $react_fields['companyValidationErrors'] = {};
                    $react_fields['showModal'] = false;

                    return $react_fields;
		},
		computed: {
                    groupFields: function () {

                        var groups = {
                            personal: {
                                keys: this.personalFields,
                                fields: [],
                                rows: [],
                            },
                            address: {
                                keys: this.addressFields,
                                fields: [],
                                rows: [],
                            },
                        };

			for (let group in groups) {
			    groups[group].keys.forEach((key) => {
				// Include company fields - visibility is controlled by v-show and the visibility property
				var companyFields = ['billing_company_vat', 'short_address', 'building_number', 'address_second', 'district', 'postal_code'];
				var isCompanyField = companyFields.indexOf(key) > -1;
				
				// For company fields, ONLY include if checkbox is checked
				if (isCompanyField) {
				    var isChecked = this.isCompanyChecked;
				    var hasField = this.fields.hasOwnProperty(key);
				    var fieldVisibility = hasField && this.fields[key].visibility;
				    
				    // Only add company fields if checkbox is checked
				    if (isChecked && hasField && fieldVisibility) {
					groups[group].fields.push(Object.assign(this.fields[key], {key: key}));
				    }
				    // Don't add company fields if checkbox is not checked
				} else {
				    // For non-company fields, use normal visibility check
				    if (this.fields.hasOwnProperty(key) && this.fields[key].visibility) {
					groups[group].fields.push(Object.assign(this.fields[key], {key: key}));
				    }
				}
			    })
			}

                        let $elementsInRow = this.countFieldsInRow;

                        for (let group in groups) {

                            let $row      = [];
                            let $numIndex = 0;

                            groups[group].fields.forEach (function (v, i) {

                                if (v.key === 'email' || v.key === 'role') {

				    $row.push(v);

				    if (v.key === 'email' && groups[group].fields[i + 1] && groups[group].fields[i + 1].key === 'role') {
					return true;
				    }

                                    groups[group].rows.push($row);

				    $row = [];

				    return true;
                                }

                                $row.push(v);

                                if ($row.length < $elementsInRow) {
                                    return true;
                                }

                                groups[group].rows.push($row);

                                $row = [];
                            });

                            if ($row.length) {
                                groups[group].rows.push($row);
                            }
                        }

                        return groups;
                    },
                    defaultCountryList () {
                        return this.$root.defaultCountriesList;
                    },
                    defaultStatesList () {
                        return this.$root.defaultStatesList;
                    },
                    googleAutocompleteAPIKey () {
                        return this.getSettingsOption('google_map_api_key');
                    },
                    countCols () {
                        return Math.floor(12/this.countFieldsInRow);
                    },
		    addressFields() {
			var addressFields = this.addressFieldsOrder;
			return addressFields.concat(this.addressAdditionalKeys);
		    },
		    addressAdditionalKeys() {
			var addressAdditionalKeys = [];
			for (let $key in this.customerAddressAdditionalKeys) {
			    if (this.customerAddressAdditionalKeys.hasOwnProperty($key)) {
				addressAdditionalKeys.push($key);
			    }
			}

			return addressAdditionalKeys;
		    },
                    personalFields () {
                        return this.personalFieldsOrder;
                    },
		    doNotSubmitOnEnterLastField() {
			    return this.getSettingsOption( 'do_not_submit_on_enter_last_field' );
		    },
		    storedCustomer() {
			return this.$store.state.add_order.cart.customer;
		    },
		    customerCustomFields() {
			return this.storedCustomer ? this.storedCustomer.custom_fields : {};
		    },
		    hideFieldsList() {
			return this.getSettingsOption('customer_hide_fields', []);
		    },
		    requiredFieldsList() {
			return this.getSettingsOption('customer_required_fields', []);
		    },
		    customerFieldListWithoutBillingShippingPrefix() {
			return ['role', 'locale', 'billing_company_checkbox'];
		    },
		    isCompanyChecked() {
			var checkboxField = this.fields['billing_company_checkbox'];
			if (!checkboxField || !checkboxField.visibility) {
			    return false;
			}
			var value = checkboxField.value;
			// Treat only explicit string/number 1 as true
			var result = value === '1' || value === 1;
			return result;
		    },
		},
        watch: {
            customerCustomFields(newVal, oldVal) {
                this.$root.bus.$emit('customer-custom-fields-at-top-set-custom-fields', {custom_fields: newVal});
                this.$root.bus.$emit('customer-custom-fields-set-custom-fields', {custom_fields: newVal});
            },
	    'fields.billing_company_checkbox.value': {
		handler(newValue) {
		    // Normalize: only literal '1' counts as checked
		    const normalized = newValue === '1' ? '1' : '';
		    if (this.fields['billing_company_checkbox']) {
			this.fields['billing_company_checkbox'].value = normalized;
		    }
		    const isCompany = normalized === '1';
		    this.updateCompanyFieldsVisibility(isCompany);
		},
		immediate: false
	    }
        },
		methods: {
			initForm( $reactFields ) {
				if ( typeof $reactFields === 'undefined' ) {
					var $reactFields = {};
				}
				$reactFields['fields'] = {};
                                $reactFields['visibleFields'] = [];
                                $reactFields['readonlyFieldsList'] = [];

				if ( typeof this.fieldsToShow === 'undefined') {
					return $reactFields;
				}

				// copy without reference
				var $fields = JSON.parse( JSON.stringify( this.fieldsToShow ) );

				var defaultValues = {
				    city: this.getSettingsOption('default_city'),
				    country: this.getSettingsOption('default_country'),
				    state: this.getSettingsOption('default_state'),
				    postcode: this.getSettingsOption('default_postcode'),
				};

				var visibilityFields = {
				    email: this.addressType === 'billing',
				    phone: this.addressType === 'billing' || this.getSettingsOption( 'use_shipping_phone', false ),
				    vat_number: this.addressType === 'billing' && this.getSettingsOption('support_field_vat', false ),
				    role: this.addressType === 'billing' && this.getSettingsOption('customer_show_role_field', false ) && +this.storedCustomer.id,
				    locale: this.addressType === 'billing' && this.getSettingsOption('customer_show_language_field', false ) && +this.storedCustomer.id,
				    billing_company_checkbox: this.addressType === 'billing',
				};
				
				// Company fields visibility - set to true so they're included in groupFields, but v-show will control display
				var companyFields = ['billing_company_vat', 'short_address', 'building_number', 'address_second', 'district', 'postal_code'];
				companyFields.forEach((fieldKey) => {
				    visibilityFields[fieldKey] = this.addressType === 'billing'; // Include in groupFields, v-show will hide them
				});

				this.hideFieldsList.forEach((hide_field) => {
				    visibilityFields[hide_field] = false;
				});

				var isEmptyCustomer = typeof this.storedCustomer.billing_first_name === 'undefined';

				for ( let $field in $fields ) {
					if ( $fields.hasOwnProperty( $field ) ) {
						$reactFields['fields'][$field] = $fields[$field];

						if (typeof $reactFields['fields'][$field]['visibility'] === 'undefined') {
						    $reactFields['fields'][$field]['visibility'] = true;
						}

						if (typeof visibilityFields[$field] !== 'undefined') {
						    $reactFields['fields'][$field]['visibility'] = visibilityFields[$field];
						}

						if (typeof defaultValues[$field] !== 'undefined' && isEmptyCustomer) {
						    $reactFields['fields'][$field]['value'] = defaultValues[$field];
						}
					}

                                        if ($reactFields['fields'][$field]['visibility']) {
                                            $reactFields['visibleFields'].push($field);
                                        }
				}

				if ( $fields.hasOwnProperty( 'country' )
                                        && typeof this.defaultCountryList !== 'undefined'
                                        && typeof this.defaultStatesList !== 'undefined'
                                ) {
					var country = this.getObjectByKeyValue( this.defaultCountryList, 'value',
						$fields['country'].value );

					var statesList = this.defaultStatesList[$fields['country'].value] || [];

					$reactFields['fields']['country']['value'] = country || {value: ''};

					//	if formToDefault() we need to update 'statesList'
					this.statesList = statesList;

					$reactFields['statesList'] = statesList;
				}

				if ( $fields.hasOwnProperty( 'state' )
                                    && typeof statesList !== 'undefined'
                                ) {
					var state = null;

					if ( statesList.length ) {
						state = this.getObjectByKeyValue( statesList, 'value', $fields['state'].value, this.getObjectByKeyValue( statesList, 'value', '' ) );
					} else {
						state = $fields['state'].value;
					}

					$reactFields['fields']['state']['value'] = state;
				}

				if ( $fields.hasOwnProperty( 'role' )
                                    && typeof this.rolesList !== 'undefined'
                                ) {
					var role = this.getObjectByKeyValue( this.rolesList, 'value',
						$fields['role'].value );

					$reactFields['fields']['role']['value'] = role;
				}

				if ( $fields.hasOwnProperty( 'locale' )
                                    && typeof this.languagesList !== 'undefined'
                                ) {
					var locale = this.getObjectByKeyValue( this.languagesList, 'value',
						$fields['locale'].value );

					$reactFields['fields']['locale']['value'] = locale;
				}

				// Handle company checkbox - convert to '1' or ''
				if ( $fields.hasOwnProperty( 'billing_company_checkbox' ) ) {
				    var checkboxValue = $fields['billing_company_checkbox'].value;
				    $reactFields['fields']['billing_company_checkbox']['value'] = (checkboxValue === '1' || checkboxValue === 1 || checkboxValue === true) ? '1' : '';
				} else {
				// If checkbox field doesn't exist, create it with default value
			    if (typeof $reactFields['fields']['billing_company_checkbox'] === 'undefined') {
					$reactFields['fields']['billing_company_checkbox'] = {
					    label: 'مؤسسة ؟',
					    value: '',
					    visibility: this.addressType === 'billing'
					};
			    }
				}
				
				// Check if any company field has a value to auto-check the checkbox
				var companyFields = ['billing_company_vat', 'short_address', 'building_number', 'address_second', 'district', 'postal_code'];
				var hasCompanyData = false;
				companyFields.forEach((fieldKey) => {
				    if ($reactFields['fields'][fieldKey] && $reactFields['fields'][fieldKey].value && $reactFields['fields'][fieldKey].value.toString().trim() !== '') {
					hasCompanyData = true;
				    }
				});
				
				// Auto-check checkbox if company data exists
				if (hasCompanyData && $reactFields['fields']['billing_company_checkbox']) {
				    $reactFields['fields']['billing_company_checkbox']['value'] = '1';
				}
				
				// Set company fields visibility based on checkbox
				// Note: We keep visibility true so they're included in groupFields, v-show will control display
				var isCompany = ($reactFields['fields']['billing_company_checkbox'] && $reactFields['fields']['billing_company_checkbox']['value'] === '1');
				// Don't change visibility here - let v-show handle it

                                let readonlyFieldsList = [];

                                for (let field in $reactFields['fields']) {
                                    if (this.readonlyFields.indexOf(field) > -1 && $reactFields['fields'][field]['value'] !== '') {
                                        readonlyFieldsList.push(field)
                                    }
                                }

                                $reactFields['readonlyFieldsList'] = readonlyFieldsList;

				return $reactFields;
			},
			formLoad() {

                            let $reactFields		    = this.initForm();
                            this.fields			    = $reactFields.fields;
                            this.visibleFields		    = $reactFields.visibleFields;
                            this.error			    = '';
                            this.error_code		    = '';
                            this.emptyRequiredFieldsList    = [];
                            
                            // Clear company validation errors on form load
                            this.companyValidationErrors = {};
                            
                            // If checkbox is not checked, ensure no company validation errors
                            if (!this.isCompanyChecked) {
				this.companyValidationErrors = {};
				// Also clear company fields from empty required fields list
				var companyFields = ['billing_company_vat', 'short_address', 'building_number', 'address_second', 'district', 'postal_code'];
				this.emptyRequiredFieldsList = this.emptyRequiredFieldsList.filter(function(field) {
				    return companyFields.indexOf(field) === -1;
				});
                            }
                            
                            this.readonlyFieldsList         = $reactFields.readonlyFieldsList;

			    this.$nextTick(() => {
				// add second $nextTick() because BModal::methods::focusFirst() set new focus
				this.$nextTick(() => {
				    if (this.visibleFields.length) {
					this.$refs[this.visibleFields[0]][0].focus();
				    }
				});
			    });
			},
			hideModal() {
			    if (typeof this.$refs.google_autocomplete !== 'undefined') {
				this.$refs.google_autocomplete.forEach((component) => {
				   component.clear();
				   component.init();
				});
			    }
			},
			isFieldVisible(fieldKey) {
			    // Company fields that should be hidden when checkbox is unchecked
			    var companyFields = ['billing_company_vat', 'short_address', 'building_number', 'address_second', 'district', 'postal_code'];
			    
			    if (companyFields.indexOf(fieldKey) > -1) {
				// Double check: both the checkbox must be checked AND the field must have visibility true
				var checkboxField = this.fields['billing_company_checkbox'];
				var isChecked = checkboxField && (checkboxField.value === '1' || checkboxField.value === 1 || checkboxField.value === true || checkboxField.value === 'true');
				var fieldVisibility = this.fields[fieldKey] && this.fields[fieldKey].visibility;
				return isChecked && fieldVisibility;
			    }
			    
			    return true;
			},
            onCompanyCheckboxChange(newValue) {
                // Normalize incoming value: only literal '1' counts as checked; everything else becomes ''
                const normalized = newValue === '1' ? '1' : '';

                if (this.fields['billing_company_checkbox']) {
                    this.fields['billing_company_checkbox'].value = normalized;
                }

                const isCompany = normalized === '1';

                this.updateCompanyFieldsVisibility(isCompany);
            },
            updateCompanyFieldsVisibility(isCompany) {
			    var companyFields = ['billing_company_vat', 'short_address', 'building_number', 'address_second', 'district', 'postal_code'];
			    
			    // Update visibility and clear values/errors based on checkbox state
			    companyFields.forEach((fieldKey) => {
				if (this.fields[fieldKey]) {
				    // Update visibility property - direct assignment works in Vue 3
				    this.fields[fieldKey].visibility = isCompany;
				    
				    // Clear values and errors when unchecked
				    if (!isCompany) {
					this.fields[fieldKey].value = '';
					if (this.companyValidationErrors[fieldKey]) {
					    delete this.companyValidationErrors[fieldKey];
					}
				    }
				}
			    });
			    
			    // Clear all company validation errors when unchecked
			    if (!isCompany) {
				this.companyValidationErrors = {};
				
				// Remove company fields from empty required fields list
				var filteredList = this.emptyRequiredFieldsList.filter(function(field) {
				    return companyFields.indexOf(field) === -1;
				});
				this.emptyRequiredFieldsList = filteredList;
				
				// Clear any error message if it was about company fields
				if (this.error) {
				    var hasCompanyError = false;
				    for (var fieldKey in this.companyValidationErrors) {
					if (this.companyValidationErrors.hasOwnProperty(fieldKey)) {
					    hasCompanyError = true;
					    break;
					}
				    }
				    if (!hasCompanyError) {
					// Check if error message contains company-related text
					var companyErrorKeywords = ['ضريبي', 'مؤسسة', 'مبنى', 'فرعي', 'حي', 'بريدي', 'مختصر'];
					var isCompanyError = companyErrorKeywords.some(function(keyword) {
					    return this.error.indexOf(keyword) !== -1;
					}.bind(this));
					if (isCompanyError) {
					    this.error = '';
					}
				    }
				}
			    }
			    
			    // Force Vue to update visibleFields
			    this.$nextTick(() => {
				this.visibleFields = Object.keys(this.fields).filter((key) => {
				    return this.fields[key].visibility;
				});
			    });
			},
			clearCompanyFieldError(fieldKey) {
			    // Clear validation error when user starts typing
			    if (this.companyValidationErrors[fieldKey]) {
				delete this.companyValidationErrors[fieldKey];
			    }
			},
			cancel() {
				this.showModal = false;
			},
			validateCompanyField(fieldKey, value) {
			    var errors = [];
			    var trimmedValue = (value !== null && value !== undefined) ? value.toString().trim() : '';
			    
			    switch(fieldKey) {
				case 'billing_company_vat':
				    if (trimmedValue === '') {
					errors.push('الرقم الضريبي للمؤسسة مفقود أو فارغ.');
				    } else {
					var digitsOnly = trimmedValue.replace(/\D/g, '');
					if (digitsOnly !== trimmedValue) {
					    errors.push('الرقم الضريبي للمؤسسة يجب أن يحتوي على أرقام فقط.');
					} else if (digitsOnly.length !== 15) {
					    errors.push('الرقم الضريبي للمؤسسة يجب أن يتكون من 15 رقمًا.');
					} else if (/^0+$/.test(digitsOnly)) {
					    errors.push('الرقم الضريبي للمؤسسة لا يمكن أن يكون مكونًا من أصفار فقط.');
					} else if (digitsOnly[0] !== '3' || digitsOnly[digitsOnly.length - 1] !== '3') {
					    errors.push('الرقم الضريبي للمؤسسة يجب أن يبدأ بالرقم 3 وينتهي بالرقم 3.');
					}
				    }
				    break;
				    
				case 'short_address':
				    if (trimmedValue === '') {
					errors.push('العنوان المختصر مطلوب عند اختيار المؤسسة.');
				    }
				    break;
				    
				case 'building_number':
				    if (trimmedValue === '') {
					errors.push('رقم المبنى مطلوب عند اختيار المؤسسة.');
				    } else {
					var digitsOnly = trimmedValue.replace(/\D/g, '');
					if (digitsOnly !== trimmedValue) {
					    errors.push('رقم المبنى يجب أن يحتوي على أرقام فقط.');
					} else if (digitsOnly.length !== 4) {
					    errors.push('رقم المبنى يجب أن يتكون من 4 أرقام.');
					} else if (/^0+$/.test(digitsOnly)) {
					    errors.push('رقم المبنى لا يمكن أن يكون مكونًا من أصفار فقط.');
					}
				    }
				    break;
				    
				case 'address_second':
				    if (trimmedValue === '') {
					errors.push('الرقم الفرعي مطلوب عند اختيار المؤسسة.');
				    } else {
					var digitsOnly = trimmedValue.replace(/\D/g, '');
					if (digitsOnly !== trimmedValue) {
					    errors.push('الرقم الفرعي يجب أن يحتوي على أرقام فقط.');
					} else if (digitsOnly.length !== 4) {
					    errors.push('الرقم الفرعي يجب أن يتكون من 4 أرقام.');
					} else if (/^0+$/.test(digitsOnly)) {
					    errors.push('الرقم الفرعي لا يمكن أن يكون مكونًا من أصفار فقط.');
					}
				    }
				    break;
				    
				case 'district':
				    if (trimmedValue === '') {
					errors.push('الحي مطلوب عند اختيار المؤسسة.');
				    }
				    break;
				    
				case 'postal_code':
				    if (trimmedValue === '') {
					errors.push('الرمز البريدي مطلوب عند اختيار المؤسسة.');
				    } else {
					var digitsOnly = trimmedValue.replace(/\D/g, '');
					if (digitsOnly !== trimmedValue) {
					    errors.push('الرمز البريدي يجب أن يحتوي على أرقام فقط.');
					} else if (digitsOnly.length !== 5) {
					    errors.push('الرمز البريدي يجب أن يتكون من 5 أرقام.');
				} else if (digitsOnly[0] === '0') {
				    errors.push('الرمز البريدي يجب ألا يبدأ بالرقم 0.');
					} else if (/^0+$/.test(digitsOnly)) {
					    errors.push('الرمز البريدي لا يمكن أن يكون مكونًا من أصفار فقط.');
					}
				    }
				    break;
			    }
			    
			    return errors;
			},
			isCustomerValid() {
			    var valid = true;

			    this.emptyRequiredFieldsList = [];
			    
			    // Check if company checkbox is checked - MUST be explicitly checked
			    var isCompany = this.isCompanyChecked;
			    
			    // ALWAYS clear company validation errors first
			    this.companyValidationErrors = {};
			    
			    // If checkbox is NOT checked, skip all company field validation
			    if (!isCompany) {
				// Remove any company fields from empty required fields list
				var companyFields = ['billing_company_vat', 'short_address', 'building_number', 'address_second', 'district', 'postal_code'];
				
				// Validate only regular required fields
				for ( let $field in this.fields ) {
				    // skip company fields entirely when unchecked
				    if (companyFields.indexOf($field) !== -1) {
					continue;
				    }
				    if (this.requiredFieldsList.indexOf($field) > -1 && this.fields[$field].visibility) {
					var value = this.fields[$field].value !== null && typeof this.fields[$field].value === 'object' ? this.fields[$field].value.value : this.fields[$field].value;
					if (value && value.toString().trim() === '') {
					    this.emptyRequiredFieldsList.push($field);
					    valid = false;
					}
				    }
				}
				
			    } else {
				// Checkbox IS checked - validate company fields
				var companyRequiredFields = ['billing_company_vat', 'short_address', 'building_number', 'address_second', 'district', 'postal_code'];

				for ( let $field in this.fields ) {
				    // Validate regular required fields
				    if (this.requiredFieldsList.indexOf($field) > -1 && this.fields[$field].visibility) {
					var value = this.fields[$field].value !== null && typeof this.fields[$field].value === 'object' ? this.fields[$field].value.value : this.fields[$field].value;
					if (value && value.toString().trim() === '') {
					    this.emptyRequiredFieldsList.push($field);
					    valid = false;
					}
				    }
				    
				    // Validate company fields ONLY when checkbox is checked
				    if (companyRequiredFields.indexOf($field) > -1 && this.fields[$field].visibility) {
					var value = this.fields[$field].value !== null && typeof this.fields[$field].value === 'object' ? this.fields[$field].value.value : this.fields[$field].value;
					var fieldErrors = this.validateCompanyField($field, value);
					
					if (fieldErrors.length > 0) {
					    this.emptyRequiredFieldsList.push($field);
					    this.companyValidationErrors[$field] = fieldErrors;
					    valid = false;
					}
				    }
				}
			    }

			    if (this.emptyRequiredFieldsList.length) {
				valid = false;
			    }

			    this.getEditCustomerValidation().forEach((validation) => {
                    if (!!!validation()) {
                        valid = false;
                        return;
                    }
			    });

			    return valid;
			},
			saveAddress() {

                            if ( ! this.customer ) {
                                    this.customer = {};
                            }

                            var customer = Object.assign({}, this.customer);

                            if ( !!! this.isCustomerValid() ) {
				// Build error message from validation errors
				var errorMessages = [];
				
				// Add company validation errors
				for (var fieldKey in this.companyValidationErrors) {
				    if (this.companyValidationErrors.hasOwnProperty(fieldKey)) {
					errorMessages = errorMessages.concat(this.companyValidationErrors[fieldKey]);
				    }
				}
				
				// Add general required fields error if there are empty required fields
				if (this.emptyRequiredFieldsList.length > 0 && errorMessages.length === 0) {
				    this.error = this.fillAllFieldsLabel;
				} else if (errorMessages.length > 0) {
				    this.error = errorMessages.join(' ');
				} else {
				    this.error = this.fillAllFieldsLabel;
				}
                                return;
                            }

                            // Map company fields to their correct keys
                            var companyFieldMapping = {
				'billing_company_checkbox': 'billing_billing_company_checkbox',
				'billing_company_vat': 'billing_billing_company_vat',
				'short_address': 'billing_short_address',
				'building_number': 'billing_building_number',
				'address_second': 'billing_address_second',
				'district': 'billing_district',
				'postal_code': 'billing_postal_code'
                            };

                            for ( let $field in this.fields ) {
                                if ( this.fields.hasOwnProperty($field) ) {

				    var $fieldKey;
				    
				    // Handle company field mapping
				    if (companyFieldMapping.hasOwnProperty($field)) {
					$fieldKey = companyFieldMapping[$field];
				    } else if (this.customerFieldListWithoutBillingShippingPrefix.indexOf($field) > -1) {
					$fieldKey = $field;
				    } else {
					$fieldKey = this.addressType + '_' + $field;
				    }

				    if ( this.fields[$field].value !== null && typeof this.fields[$field].value === 'object' && this.fields[$field].value.hasOwnProperty( 'value' ) ) {
					customer[$fieldKey] = this.fields[$field].value.value; // for multiselect fields
				    } else {
					customer[$fieldKey] = this.fields[$field].value;
				    }
                                }
                            }

			    customer['custom_fields'] = Object.assign( {}, this.customFieldsAtTop, this.customFields );

			    this.validateAddressByUSPS({
				street1: customer[this.addressType + '_address_1'],
				street2: customer[this.addressType + '_address_2'],
				city: customer[this.addressType + '_city'],
				state: customer[this.addressType + '_state'],
				zip: customer[this.addressType + '_postcode'],
				country: customer[this.addressType + '_country'],
			    }, (address) => {

				var validatedFields = {
				    address_1: address.street1,
				    address_2: address.street2,
				    city: address.city,
				    state: address.state,
				    postcode: address.zip,
				};

				this.updateAddress(validatedFields);

				for (let field in validatedFields) {
				    customer[this.addressType + '_' + field] = validatedFields[field];
				}

				this.updateCustomerRequest(customer, (response) => {

				    let $data = response.data.data;

				    if ( response.data.success === true ) {
					this.updateCustomer($data.customer);
					    this.showModal = false;
				    } else {
					this.error	= $data.message;
					this.error_code = $data.code;
				    }
				});

			    }, (error) => {
				this.error = error;
			    });

			},
			updateCustomerRequest(customer, callback, params) {

				let $args = Object.assign({
					customer_data: customer,
					action: 'phone-orders-for-woocommerce',
					method: 'update_customer',
					_wp_http_referer: this.referrer,
					_wpnonce: this.nonce,
					tab: this.tabName,
                                        nonce: this.nonce,
				}, params || {});

                                this.error	= '';
                                this.error_code = '';

				var formData = new FormData();
				this.buildFormData(formData, $args);

				this.axios.post( this.url, formData, { headers: { 'Content-Type': 'multipart/form-data' } } ).then( ( response ) => {
				    callback(response);
				}, () => {} );
			},
			onChangeDefaultCountry: function (fieldKey) {
				this.fields.state.value = '';
				this.statesList = this.defaultStatesList[this.fields.country.value.value] || [];

				if ( this.statesList.length ) {
					this.fields.state.value = this.getObjectByKeyValue( this.statesList, 'value', this.fields.state.value );
				}

                                this.$nextTick(() => {
                                    this.onEnter(fieldKey);
                                })
			},
			onEnter: function ( fieldKey ) {

				var currentIndex = this.visibleFields.indexOf( fieldKey );
				var nextIndex = currentIndex + 1;

				if ( typeof this.visibleFields[nextIndex] !== 'undefined' ) {

					if ( typeof this.$refs[this.visibleFields[nextIndex]][0].activate === 'function' ) {
						this.$refs[this.visibleFields[nextIndex]][0].activate();
					} else {
						this.$refs[this.visibleFields[nextIndex]][0].focus();
					}

					return;
				}

				if ( ! this.doNotSubmitOnEnterLastField ) {
					this.saveAddress();
				}

			},
                        updateAddress: function (fields) {

                            for (let fieldKey in fields) {
                                if (this.fields[fieldKey]) {

                                    this.fields[fieldKey].value = null;

                                    if (this.visibleFields.indexOf(fieldKey) > -1 || fieldKey === 'country' && this.visibleFields.indexOf('state') > -1) {
                                        this.fields[fieldKey].value = fields[fieldKey];
                                        if (fieldKey === 'country') {
                                            this.fields[fieldKey].value = this.getObjectByKeyValue(
                                                this.defaultCountryList,
                                                'value',
                                                this.fields[fieldKey].value
                                            );
                                            this.statesList = this.defaultStatesList[this.fields.country.value.value] || [];
                                        }
                                    } else {
                                        // ignore visibility for additional customer fields
                                        if (this.addressAdditionalKeys.indexOf(fieldKey) !== -1) {
                                            this.fields[fieldKey].value = fields[fieldKey];
                                        }
                                    }
                                }
                            }

                            if ( this.statesList.length ) {
                                this.fields.state.value = this.getObjectByKeyValue( this.statesList, 'value', this.fields.state.value ) || this.getObjectByKeyValue( this.statesList, 'title', this.fields.state.value ) ;
                            }
                        },
                        copyFromBillingAddress() {

			    var skipFields = ['email'];

			    if (!this.fields['phone']['visibility']) {
				skipFields.push('phone');
			    }

			    for (var fieldKey in this.fields) {

				if (skipFields.indexOf(fieldKey) === -1) {

                                    var $fieldKey = this.customerFieldListWithoutBillingShippingPrefix.indexOf(fieldKey) > -1 ? fieldKey : 'billing_' + fieldKey;

				    var value = this.customer[$fieldKey];

				    if ( fieldKey === 'country'
                                        && typeof this.defaultCountryList !== 'undefined'
                                        && typeof this.defaultStatesList !== 'undefined'
				    ) {
					value = this.getObjectByKeyValue( this.defaultCountryList, 'value', value );

					value = value || {value: ''};

					var statesList = this.defaultStatesList[value.value] || [];

					//	if formToDefault() we need to update 'statesList'
					this.statesList = statesList;
				    }

				    if ( fieldKey === 'state'
					&& typeof statesList !== 'undefined'
				    ) {
					if ( statesList.length ) {
					    value = this.getObjectByKeyValue( statesList, 'value', value );
					}
				    }

                                    if ( fieldKey === 'role'
                                        && typeof this.rolesList !== 'undefined'
                                    ) {
                                            value = this.getObjectByKeyValue( this.rolesList, 'value',
                                                    value );
                                    }

                                    if ( fieldKey === 'locale'
                                        && typeof this.languagesList !== 'undefined'
                                    ) {
                                            value = this.getObjectByKeyValue( this.languagesList, 'value',
                                                    value );
                                    }

				    this.fields[fieldKey]['value'] = value;
				}
			    }

                        },
			buildFormData(formData, data, parentKey) {
			    if (data && typeof data === 'object' && !(data instanceof Date) && !(data instanceof File)) {
			      Object.keys(data).forEach(key => {
				this.buildFormData(formData, data[key], parentKey ? `${parentKey}[${key}]` : key);
			      });
			    } else {
			      const value = data == null ? '' : data;

			      formData.append(parentKey, value);
			    }
			},
		},
		components: {
                    Multiselect,
                    GoogleAutocomplete,
                    CustomAutocomplete,
		},
	}
</script>
