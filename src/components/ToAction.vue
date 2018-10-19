<template>
    <div>
        <div class="d-flex justify-content-between mb-2">
            <b-dropdown variant="dark" :text="translate('dashboard.filteredViews')">
                <b-dropdown-item-button v-for="(pvItem) in
                filteredViews.personalViews" @click="activateFilter(pvItem)" :key="pvItem.filter_id">{{
                pvItem.name }}</b-dropdown-item-button>
            </b-dropdown>
            <div>
                <b-dropdown v-if="this.savedView" variant="dark" @click="saveViewModal" split :text="translate('dashboard.saveFilterAs')">
                    <b-dropdown-item-button @click="renameViewModal" v-lang.general.rename></b-dropdown-item-button>
                    <b-dropdown-item-button @click="deleteViewModal" class="text-danger" v-lang.general.delete></b-dropdown-item-button>
                </b-dropdown>
                <b-btn v-else-if="this.activeFilters.length" variant="dark" @click="saveViewModal" v-lang.dashboard.saveFilter></b-btn>
                <b-btn v-if="showPolicyCreateButton" @click="onCreatePolicyClick" variant="dark" class="ml-2" v-lang.dashboard.policy.createPolicy></b-btn>
            </div>
        </div>

        <div class="nt-pull-up">

                <b-form-row>
                    <b-col cols="7" lg="5" class="mb-3 mb-lg-0">
                        <risk-rating-slider></risk-rating-slider>
                    </b-col>
                    <b-col cols="4" lg="2" class="mb-3 mb-lg-0">
                        <label v-lang.dashboard.riskType></label>
                        <risk-type :riskTypes="alphabeticalRiskCategories"></risk-type>
                    </b-col>
                    <b-col cols="5" lg="3">
                        <label v-lang.dashboard.country></label>
                        <country-filter :countries="countriesComputed"></country-filter>
                    </b-col>
                    <b-col cols="4" lg="2" v-if="isPowImplemented">
                        <label v-lang.dashboard.realBrowser></label>
                        <real-browser-filter :realBrowserOptions="realBrowserOptionValues"></real-browser-filter>
                    </b-col>
                    <b-col cols="4" lg="2">
                        <label v-lang.dashboard.groups></label>
                        <group-label-filter :groupLabels="groupLabelsComputed"></group-label-filter>
                    </b-col>
                </b-form-row>

                <div slot="active-filters">
                    <div class="d-flex flex-wrap align-items-center">
                        <h6 class="mr-2">
                            <span class="text-muted">{{ currentFilterTitle }}</span>:
                        </h6>
                        <div v-if="sliderActiveFilter">
                            <b-badge pill variant="dark" class="mr-2">
                                <ul>
                                    <li v-if="riskRatingMin < 20 && riskRatingMax >= 20" class="td-dot td-dot--badge td-dot--lowest"></li>
                                    <li v-if="riskRatingMin <= 20 && riskRatingMax >= 40" class="td-dot td-dot--badge td-dot--low"></li>
                                    <li v-if="riskRatingMin <= 40 && riskRatingMax >= 60" class="td-dot td-dot--badge td-dot--medium"></li>
                                    <li v-if="riskRatingMin <= 60 && riskRatingMax >= 80" class="td-dot td-dot--badge td-dot--high"></li>
                                    <li v-if="riskRatingMin <= 80 && riskRatingMax > 80" class="td-dot td-dot--badge td-dot--highest"></li>
                                    <li v-if="showFilterEdit" @click="resetSlider" class="pointer fa fa-times pl-1"></li>
                                </ul>
                            </b-badge>
                        </div>

                        <b-badge pill variant="dark" v-for="(cFilter, cIndex) in combinedSelectedFilter" :key="cIndex" class="pointer mr-1">{{ mapRiskCategoriesToLabels(cFilter) }}<span v-if="showFilterEdit" @click="clearFilter(cFilter)" class="fa fa-times pl-1"></span></b-badge>
                        <b-btn v-if="showFilterEdit" variant="link" size="sm" @click="clearAllFilters" v-lang.general.clear></b-btn>

                        <b-modal id="modalSave" ref="modalSave" size="sm" :title="translate('dashboard.saveView')" :ok-title="translate('general.save')" @ok="saveView" @hidden="cancelSaveView">
                            <b-form-group :label="translate('general.name')" label-for="inputSave" class="mb-0">
                                <b-input id="inputSave" ref="inputSave" v-model="inputSaveValue" :state="stateError(!$v.inputSaveValue.$error)" @input="$v.inputSaveValue.$touch()" :placeholder="translate('dashboard.giveViewName')"></b-input>
                                <b-form-invalid-feedback>
                                    <required-validation :validation="$v.inputSaveValue"></required-validation>
                                    <max-length-validation :validation="$v.inputSaveValue"></max-length-validation>
                                </b-form-invalid-feedback>
                            </b-form-group>
                        </b-modal>

                        <b-modal id="modalRename" ref="modalRename" size="sm" :title="translate('dashboard.renameView')" :ok-title="translate('general.save')" @ok="renameEditView">
                            <b-form-group :label="translate('general.name')" label-for="inputRename">
                                <b-input id="inputRename" ref="inputRename" v-model="inputRenameValue" :state="stateError(!$v.inputRenameValue.$error)" @input="$v.inputRenameValue.$touch()" :placeholder="translate('dashboard.renameThisView')"></b-input>
                                <b-form-invalid-feedback>
                                    <required-validation :validation="$v.inputRenameValue"></required-validation>
                                    <max-length-validation :validation="$v.inputRenameValue"></max-length-validation>
                                </b-form-invalid-feedback>
                            </b-form-group>
                        </b-modal>

                        <b-modal id="modalDelete" ref="modalDelete" size="sm" :title="translate('dashboard.deleteView')" :ok-title="translate('general.delete')" ok-variant="danger" @ok="deleteView">
                            {{ translate('dashboard.deleteViewMessage') }}
                        </b-modal>

                        <b-modal id="modalDiscardChanges" ref="modalDiscardChanges" size="sm" :title="translate('general.discardChanges')" :ok-title="translate('general.discardChanges')" @ok="discardChanges">
                            {{ translate('dashboard.discardChangesMessage') }}
                        </b-modal>

                        <b-modal id="modalSaveChanges" ref="modalSaveChanges" size="sm" :title="translate('general.saveChanges')" :ok-title="translate('general.save')" @ok="saveChanges">
                            <p>{{ translate('dashboard.saveChangesMessage') }}</p>
                            <p class="mb-0">{{ translate('dashboard.areYouSure') }}</p>
                        </b-modal>
                    </div>
                </div>
            <b-modal id="errorModal" ref="errorModal" hide-header size="sm">
                <p>{{ errorMessage }}</p>
                <div slot="modal-footer">
                    <b-btn @click="closeErrorModal" v-lang.general.close></b-btn>
                </div>
            </b-modal>
        </div>
    </div>
</template>

<script>

export default {
  data() {
    return {
      errorMessage: '',
      countries: [],
      realBrowserOptions: '',
      fields: [],
      filterSearch: null,
      savedView: false,
      editedView: false,
      name: '',
      inputSaveValue: '',
      inputEditValue: '',
      inputRenameValue: '',
      uniqueCategories: [],
      prevId: null,
      showCollapse: this.$route.params.showCreatePolicy || false,
      showFilterEdit: true
    }
  },
  computed: {
    ...mapGetters({
      topHundredThreats: 'topHundredThreats',
      filters: 'filters',
      botManagement: 'botManagement',
      userDetails: 'userDetails',
      visitorFilters: 'visitorFilters',
      currentSavedFiltersId: 'currentSavedFiltersId',
      currentSavedFiltersName: 'currentSavedFiltersName',
      selectedDomainAddress: 'selectedDomainAddress',
      botManagementPermissions: 'permissions/botManagement'
    }),
    showPolicyCreateButton() {
      return this.policyPermissions.update && this.botManagement.featureToggles.policiesEnabled
    },
    policyPermissions() {
      const { policy } = this.botManagementPermissions

      return object.isEmpty(policy) ? { read: false, update: false } : policy
    },
    filteredViews() {
      return { personalViews: this.visitorFilters }
    },
    currentFilterTitle() {
      return this.currentSavedFiltersName || this.translate('dashboard.currentFilter')
    },
    alphabeticalRiskCategories() {
      let entries = []
      if (this.topHundredThreats.length > 0) {
        const categories = this.topHundredThreats.map(visitor => Object.keys(visitor.riskCategories))
        const flattened = [...new Set(categories.flat())]
        entries = flattened
          .map(category => {
            return {
              value: category,
              label: utils.mapRiskCategoriesToLabels(category)
            }
          })
          .sort(sort.object.alphabeticalDescending('label'))
      }
      return entries
    },
    groupLabelsComputed() {
      let entries = []
      if (this.topHundredThreats.length > 0) {
        const groups = this.topHundredThreats.map(visitor => visitor.myPathsLabel)
        const flattened = [...new Set(groups.flat())]
        entries = flattened
          .map(label => {
            return {
              value: label,
              label: utils.mapPathGroupsToLabels(label)
            }
          })
          .sort(sort.object.alphabeticalAscending('label'))
      }
      return entries
    },
    riskRatingMin() {
      return this.filters.riskRating.min
    },
    riskRatingMax() {
      return this.filters.riskRating.max
    },
    sliderActiveFilter() {
      return this.riskRatingMin !== 0 || this.riskRatingMax !== 100
    },
    sliderRange() {
      return this.sliderActiveFilter ? [this.riskRatingMin, this.riskRatingMax].join() : []
    },
    selectedCountries() {
      return this.filters.selectedCountryFilters || []
    },
    selectedGroupLabels() {
      return this.filters.selectedGroupLabelFilters || []
    },
    selectedRiskTypes() {
      return this.filters.selectedRiskTypeFilters || []
    },
    realBrowserPill() {
      return this.selectedRealBrowser
    },
    combinedSelectedFilter() {
      return [
        ...this.selectedCountries,
        ...this.selectedRiskTypes,
        ...this.realBrowserPill,
        ...this.selectedGroupLabels
      ]
    },
    selectedRealBrowser() {
      return this.filters.selectedRBOption !== '' && this.filters.selectedRBOption !== undefined
        ? [this.filters.selectedRBOption]
        : []
    },
    activeFilters() {
      return [
        ...this.sliderRange,
        ...this.selectedCountries,
        ...this.selectedRealBrowser,
        ...this.selectedRiskTypes,
        ...this.selectedGroupLabels
      ]
    },
    filterViews() {
      return this.filterView || []
    },
    currentFilter() {
      if (!this.activeFilters.length) {
        return null
      }
      return {
        countries: this.selectedCountries,
        risk: {
          types: this.selectedRiskTypes,
          score: {
            min: this.riskRatingMin || 0,
            max: this.riskRatingMax || 100
          }
        },
        browser: this.realBrowserPill,
        mypaths_label: this.selectedGroupLabels
      }
    },
    fieldsComputed() {
      return [
        {
          key: 'highestThreatScore',
          label: this.translate('dashboard.tableHeaderRiskRating'),
          sortable: true,
          class: 'risk-rating-column'
        },
        {
          key: 'client',
          label: this.translate('dashboard.tableHeaderVisitorID'),
          sortable: true
        },
        {
          key: 'country',
          label: this.translate('dashboard.country'),
          sortable: true
        },
        {
          key: 'dataCentre',
          label: this.translate('dashboard.tableHeaderSource'),
          sortable: true
        },
        {
          key: 'multIdentifierCounts',
          formatter: value => {
            const { ips, userAgents } = value
            if (ips === 1 && userAgents < 2) {
              return this.translate('general.no')
            }

            let ipLabel =
              ips > 1 ? this.translate('dashboard.ipAddressesLabel') : this.translate('dashboard.ipAddressLabel')
            let uaLabel =
              userAgents > 1 ? this.translate('dashboard.userAgentsLabel') : this.translate('dashboard.userAgentLabel')
            return `${ips} ${ipLabel}, ${userAgents} ${uaLabel}`
          },
          label: this.translate('dashboard.tableHeaderMultiple')
        },
        {
          key: 'bandwidth',
          formatter: value => {
            return utils.calculateDisplayedBandwidth(value)
          },
          label: this.translate('dashboard.bandwidth'),
          sortable: true
        },
        {
          key: 'blacklistReasons',
          formatter: value => {
            return value.length > 0 ? [...new Set(value)].join(', ') : this.translate('dashboard.notListed')
          },
          label: this.translate('dashboard.tableHeaderReputation'),
          sortable: true
        },
        {
          key: 'pow',
          formatter: value => {
            if (!this.isPowImplemented) {
              return this.translate('general.na')
            }
            return this.translate(`general.${value ? 'yes' : 'no'}`)
          },
          label: this.translate('dashboard.tableHeaderRealBrowser'),
          sortable: true
        },
        {
          key: 'select',
          label: '&nbsp;',
          sortable: false
        }
      ]
    },
    isPowImplemented() {
      return this.botManagement.featureToggles.powImplemented
    },
    countriesComputed() {
      let currentData = this.topHundredThreats

      currentData = currentData.map(item => {
        return item.country
      })

      currentData = [...new Set(currentData)]

      return currentData.sort(sort.basic.alphabeticalAscending)
    },
    realBrowserOptionValues() {
      let currentData = this.topHundredThreats.map(item => {
        return item.pow
      })
      return [...new Set(currentData)]
    },
    tableData() {
      let currentData = this.topHundredThreats

      // Filter on applicable risk rating and risk category (if selected)
      currentData = currentData.filter(item => {
        let applicableRiskRating

        if (this.selectedRiskTypes.length > 0) {
          const applicableRiskCategory = Object.keys(item.riskCategories).find(riskCategory => {
            return riskCategory === this.selectedRiskTypes[0]
          })

          // No matching risk category found within data
          if (typeof applicableRiskCategory === 'undefined') {
            return false
          }

          applicableRiskRating = item.riskCategories[applicableRiskCategory]
        } else {
          applicableRiskRating = item.highestThreatScore
        }

        applicableRiskRating = this.riskRatingConvert(applicableRiskRating)
        return applicableRiskRating >= this.riskRatingMin && applicableRiskRating <= this.riskRatingMax
      })

      // Filter on countries selected
      if (this.selectedCountries.length > 0) {
        currentData = currentData.filter(item => {
          return this.selectedCountries.some(country => {
            return country === item.country
          })
        })
      }

      if (this.selectedRealBrowser && this.selectedRealBrowser.length > 0) {
        currentData = currentData.filter(item => {
          return this.selectedRealBrowser[0] === item.pow
        })
      }

      if (this.selectedGroupLabels && this.selectedGroupLabels.length > 0) {
        currentData = currentData.filter(item => {
          return item.myPathsLabel.some(label => label === this.selectedGroupLabels[0])
        })
      }

      return currentData
    }
  },
  watch: {
    currentSavedFiltersId(value) {
      if (value && value.length > 0) {
        this.savedView = true
        this.editedView = false
      }
    },
    selectedDomainAddress() {
      this.clearAllFilters()
    },
    currentFilter() {
      if (!this.currentSavedFiltersId || this.prevId === this.currentSavedFiltersId) {
        this.editedView = true
      } else {
        this.editedView = false
      }
      this.prevId = this.currentSavedFiltersId
    }
  },
  methods: {
    ...mapActions({
      RESET_FILTERS_RISK_RATING: 'RESET_FILTERS_RISK_RATING',
      REMOVE_FILTERS_INDIVIDUAL_COUNTRY: 'REMOVE_FILTERS_INDIVIDUAL_COUNTRY',
      REMOVE_FILTERS_INDIVIDUAL_RISK_TYPE: 'REMOVE_FILTERS_INDIVIDUAL_RISK_TYPE',
      REMOVE_FILTERS_REAL_BROWSER: 'REMOVE_FILTERS_REAL_BROWSER',
      REMOVE_FILTERS_INDIVIDUAL_GROUP_LABEL: 'REMOVE_FILTERS_INDIVIDUAL_GROUP_LABEL',
      RESET_FILTERS_NAME: 'RESET_FILTERS_NAME',
      RESET_FILTERS_ID: 'RESET_FILTERS_ID',
      RESET_FILTERS: 'RESET_FILTERS',
      STORE_FILTERS_NAME: 'STORE_FILTERS_NAME',
      STORE_NEW_ACTIVE_FILTER: 'STORE_NEW_ACTIVE_FILTER',
      LOAD_POLICY_ACTIONS: 'policies/LOAD_POLICY_ACTIONS'
    }),
    mapRiskCategoriesToLabels(value) {
      if (typeof value === 'boolean') {
        return this.translate(`general.${value ? 'yes' : 'no'}`)
      }

      if (this.selectedRiskTypes.some(riskType => riskType === value)) {
        return utils.mapRiskCategoriesToLabels(value)
      }
      if (this.selectedGroupLabels.some(label => label === value)) {
        return utils.mapPathGroupsToLabels(value)
      }

      return value
    },
    activateFilter(item) {
      let originalFilter
      if (this.editedView) {
        originalFilter = this.visitorFilters.filter(e => {
          return e.filter_id === item.filter_id
        })[0]
      } else {
        originalFilter = item
      }
      this.RESET_FILTERS()
      this.STORE_NEW_ACTIVE_FILTER(JSON.parse(JSON.stringify(originalFilter)))
      this.editedView = false
    },
    showError(message) {
      this.errorMessage = message
      this.$refs.errorModal.show()
    },
    closeErrorModal() {
      this.$refs.errorModal.hide()
    },
    saveViewModal() {
      this.$refs.modalSave.show()
      this.focusRef(this.$refs.inputSave)
    },
    onCreatePolicySelect(filter) {
      if (filter && filter.filter_id) {
        this.activateFilter(filter)
      } else {
        this.clearAllFilters()
      }

      this.showFilterEdit = filter && !filter.filter_id
    },
    onCreatePolicyClose() {
      this.showCollapse = false
      this.showFilterEdit = true
    },
    onCreatePolicyClick() {
      if (this.editedView) {
        this.RESET_FILTERS_ID()
        this.prevId = null
      }

      this.showFilterEdit = !this.currentSavedFiltersId
      this.showCollapse = true
    },
    saveView(e) {
      this.$v.inputSaveValue.$touch()
      if (!this.currentFilter || this.$v.inputSaveValue.$invalid) {
        e.preventDefault()
      } else {
        const email = this.userDetails.key
        const name = this.inputSaveValue
        const filter = this.currentFilter
        filterService
          .addNewFilter(email, filter, name)
          .then(() => {
            filterService.getAllFilters(this.userDetails.key)
          })
          .catch(error => {
            logging.logError({ message: 'error adding new saved filter', error })
            this.showError(`${error.message}`)
          })
        this.savedView = true
        this.editedView = false
        this.$refs.modalSave.hide()
      }
    },
    cancelSaveView() {
      this.inputSaveValue = ''
      this.$nextTick(() => {
        this.$v.$reset()
      })
    },
    renameViewModal(e) {
      this.$refs.modalRename.show()
      this.focusRef(this.$refs.inputRename)
      this.inputRenameValue = this.currentSavedFiltersName
    },
    renameEditView(e) {
      this.$v.inputRenameValue.$touch()
      if (this.$v.inputRenameValue.$invalid) {
        e.preventDefault()
      } else {
        const name = this.inputRenameValue
        filterService
          .updateFilterName(this.currentSavedFiltersId, name)
          .then(() => {
            filterService.getAllFilters(this.userDetails.key)
            this.STORE_FILTERS_NAME(name)
          })
          .catch(error => {
            logging.logError({ message: 'error updating filter name', error })
            this.showError(`${error.message}`)
          })
        this.editedView = false
        this.$refs.modalRename.hide()
      }
    },
    cancelEditView() {
      this.$nextTick(() => {
        this.$v.$reset()
      })
    },
    saveChangesModal() {
      this.$refs.modalSaveChanges.show()
    },
    saveChanges() {
      this.editedView = false
    },
    deleteViewModal() {
      this.$refs.modalDelete.show()
    },
    deleteView() {
      this.savedView = false
      this.editedView = false
      filterService
        .deleteFilter(this.currentSavedFiltersId)
        .then(response => {
          filterService
            .getAllFilters(this.userDetails.key)
            .then(() => {
              this.clearAllFilters()
            })
            .catch(error => {
              logging.logError({ message: 'error getting all saved filters after delete', error })
              this.showError(`${error.message}`)
            })
        })
        .catch(error => {
          logging.logError({ message: 'error getting all saved filters after delete', error })
          this.showError(`${error.message}`)
        })
    },
    discardChangesModal() {
      this.$refs.modalDiscardChanges.show()
    },
    discardChanges() {
      this.savedView = true
      this.editedView = false
    },
    focusRef(ref) {
      this.$nextTick(() => {
        this.$nextTick(() => {
          ;(ref.$el || ref).focus()
        })
      })
    },
    riskRatingConvert(rating) {
      if (rating >= 0.95) {
        return 81
      } else if (rating >= 0.9) {
        return 61
      } else if (rating >= 0.5) {
        return 41
      } else if (rating >= 0.2) {
        return 21
      } else if (rating < 0.2) {
        return 19
      } else {
        return ''
      }
    },
    resetSlider() {
      this.RESET_FILTERS_RISK_RATING()
    },
    clearCountries(cIndex) {
      this.REMOVE_FILTERS_INDIVIDUAL_COUNTRY(cIndex)
    },
    clearFilter(cFilter) {
      this.editedView = true
      if (this.selectedCountries.indexOf(cFilter) > -1) {
        this.clearCountries(this.selectedCountries.indexOf(cFilter))
      } else if (this.selectedRiskTypes.indexOf(cFilter) > -1) {
        this.clearRiskTypes(this.selectedRiskTypes.indexOf(cFilter))
      } else if (this.selectedRealBrowser.indexOf(cFilter) > -1) {
        this.clearRealBrowser()
      } else if (this.selectedGroupLabels.indexOf(cFilter) > -1) {
        this.clearGroupLabels(this.selectedGroupLabels.indexOf(cFilter))
      }
    },
    clearRiskTypes(rIndex) {
      this.REMOVE_FILTERS_INDIVIDUAL_RISK_TYPE(rIndex)
    },
    clearGroupLabels(rIndex) {
      this.REMOVE_FILTERS_INDIVIDUAL_GROUP_LABEL(rIndex)
    },
    clearRealBrowser() {
      this.REMOVE_FILTERS_REAL_BROWSER()
    },
    clearAllFilters() {
      this.filterSearch = null
      this.RESET_FILTERS_NAME()
      this.RESET_FILTERS_ID()
      this.RESET_FILTERS()
      this.savedView = false
    },
    stateError(value) {
      return value ? null : false
    }
  },
  mixins: [validationMixin],
  validations: {
    inputSaveValue: {
      required,
      maxLength: maxLength(30)
    },
    inputEditValue: {
      required,
      maxLength: maxLength(30)
    },
    inputRenameValue: {
      required,
      maxLength: maxLength(30)
    }
  },
  mounted() {
    if (!this.isPowImplemented) {
      this.clearRealBrowser()
    }
    this.fields = this.fieldsComputed
    this.LOAD_POLICY_ACTIONS()
    filterService.getAllFilters(this.userDetails.key).catch(error => {
      logging.logError({ message: 'error getting all saved filters on mount', error })
      this.showError(`${error.message}`)
    })
  }
}
</script>

<style lang="scss" scoped>
.nt-pull-up {
  margin-top: rem(-39px);
}

.slider-wrapper {
  padding-bottom: 6px;
  padding-top: 6px;
  width: 350px;
}

/deep/ .risk-rating-column {
  width: 120px;
}
</style>
