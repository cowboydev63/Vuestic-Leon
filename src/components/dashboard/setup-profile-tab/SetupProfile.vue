<template>
  <div class="form-wizard-page mt-4">
    <div class="row">
      <div class="col-md-12">
        <vuestic-wizard
          ref="wizard"
          wizard-layout="horizontal"
          wizard-type="simple"
          :steps="hsSteps"
          class="setup-profile"
        >
          <div slot="page1" class="form-wizard-tab-content">
            <Step1 ref="registerStepOne"></Step1>
          </div>
          <div slot="page2" class="form-wizard-tab-content">
            <Step2 ref="registerStepTwo"></Step2>
          </div>
          <div slot="page3" class="form-wizard-tab-content">
            <Step3 ref="registerStepThree"></Step3>
          </div>
          <div slot="page4" class="form-wizard-tab-content">
            <Step4 ref="registerStepFour"></Step4>
          </div>
        </vuestic-wizard>
      </div>
    </div>
  </div>
</template>

<script>
import Step1 from './steps/Step1'
import Step2 from './steps/Step2'
import Step3 from './steps/Step3'
import Step4 from './steps/Step4'
import { mapGetters } from 'vuex'
import Proxy from '@/proxies/Proxy'

export default {
  name: 'setup-profile',
  components: {
    Step1,
    Step2,
    Step3,
    Step4,
  },
  watch: {
    notification: function (newVal, oldVal) {
      if (newVal.title !== 'CARD ERROR') {
        this.$refs.wizard.resetWizard()
        let that = this.$refs.registerStepOne
        that.$data.companyName = ''
        that.$data.userEmailAddress = ''
        Object.keys(that.formFields).map(field => {
          that.validateFormField(field)
        })
      }
      this.$store.dispatch('auth/notificationClear')
    }
  },
  data () {
    return {
      nextBtn: null,
    }
  },
  mounted () {
    this.nextBtn = this.$refs.wizard.nextBtn
  },
  computed: {
    hsSteps () {
      return [
        {
          label: 'Activate MINDBODY',
          slot: 'page1',
          isValid: async () => {
            this.nextBtn.loading = true
            // validation check
            const {mindbodyActivationLink, ...providerData} = this.provider
            const {success, error} = await new Proxy('checkSiteActivation.php?').submit('post', providerData)
            if (success) {
              const locations = await new Proxy('getLocations.php?').submit('post', providerData)
              this.$store.commit('auth/FORMDATA', {key: 'locations', 'value': locations})
              this.nextBtn.loading = false
              return true
            } else {
              this.$store.dispatch('auth/notification', {
                type: 'ERROR',
                title: 'Not found Account',
                message: error
              })
              this.nextBtn.loading = false
              return false
            }
          }
        },
        {
          label: 'MINDBODY location',
          slot: 'page2',
          isValid: async () => {
            this.nextBtn.loading = true
            const {locations} = this.formData
            const {mindbodyActivationLink, siteId, ...providerData} = this.provider
            const {success, error} = await new Proxy('saveLocations.php?').submit('post', {locations, providerData})
            if (success) {
              const {pricings, success, error} = await new Proxy('getPricings.php?').submit('post', providerData)
              if (success) {
                this.$store.commit('auth/FORMDATA', {key: 'pricings', value: pricings})
              } else {
                this.showToast(error)
              }
              this.nextBtn.loading = false
              return true
            } else {
              this.showToast(error)
              this.nextBtn.loading = false
              return false
            }
          }
        },
        {
          label: 'Pricing',
          slot: 'page3',
          isValid: async () => {
            this.nextBtn.loading = true
            const {pricings} = this.formData
            const {mindbodyActivationLink, siteId, ...providerData} = this.provider
            const {success, error} = await new Proxy('savePricings.php?').submit('post', {pricings, providerData})

            if (success) {
              this.nextBtn.loading = false
              return true
            } else {
              this.showToast()
              console.log(error)
              this.nextBtn.loading = false
              return false
            }
          }
        },
        {
          label: 'Payment',
          slot: 'page4',
          onNext: () => {
            // manual validation occur
            const that = this.$refs.registerStepFour
            Object.keys(that.formFields).map(field => {
              that.validateFormField(field)
            })
          },
          isValid: async () => {
            let that = this.$refs.registerStepFour
            Object.keys(that.formFields).map(field => {
              that.validateFormField(field)
            })
            // validation check
            const validOk = Object.keys(that.formFields).every(field => {
              return that.isFormFieldValid(field)
            })

            if (validOk) {
              const data = that.$data
              const {mindbodyActivationLink, siteId, ...providerData} = this.provider
              const {success, error} = await new Proxy('savePayment.php?').submit('post', {data, providerData})

              if (success) {
                this.$store.dispatch('auth/notification', {
                  type: 'INFO',
                  title: 'Conguratulation!',
                  message: 'Your account was activated'
                })
                this.$store.commit('auth/COMPLETE_SETUP_PROFILE')
                return true
              } else {
                this.showToast()
                console.log(error)
                return false
              }
            }

            return false
          }
        }
      ]
    },
    ...mapGetters({
      notification: 'auth/notificationInfo',
      formData: 'auth/getFormData',
      provider: 'auth/provider'
    })
  },
  methods: {
    showToast (errMessage = 'Oops, Please try again later.') {
      this.$store.dispatch('auth/notification', {
        type: 'ERROR',
        title: 'SERVER ERROR',
        message: errMessage
      })
    }
  }
}
</script>

<style lang="scss" scoped>
/deep/.wizard.horizontal.setup-profile {
  .wizard-body {
    padding: 2.25rem 5%;
    @include media-breakpoint-down(sm) {
      padding: $widget-padding 5%;
    }
  }
}

.form-wizard-tab-content-text {
  width: 100%; // IE11 only
}

/deep/.register-form {
  min-height: 390px;
  width: 100%;
}
</style>