<template>
  <!-- Main Wrapper -->
  <div class="main-wrapper mr-4">
    <v-alert
      v-if="alertVisible"
      position="absolute"
      location="top center"
      color="red-lighten-3"
      icon="$error"
      style="z-index: 3000; margin-top: 15px"
      closable
    >
      <div v-html="message"></div>
    </v-alert>
    <!-- Page Content -->
    <div class="row">
      <div class="col-md-12">
        <div class="appointment-tab">
          <!-- Filters -->
          <div class="flex-wrap flex-column flex-xxl-row gap-3 nav nav-tabs nav-tabs-solid pb-2">
            <div class="d-flex flex-wrap flex-column flex-lg-row gap-3 flex-auto order-2 order-xxl-1">
              <div class="flex-auto" style="width: 15rem">
                <a-select v-if="dateFilterType === 'span'"
                  v-model:value="selectedSpan"
                  style="width: 100%; align-items: center; max-height: 62px; text-align: center"
                  :options="spans"
                  size="large"
                  @change="(value) => {console.log(spanToDate(value))}"
                ></a-select>
                <a-date-picker v-if="dateFilterType === 'single'"
                  v-model:value="selectedDate"
                  format="D/M/YY"
                  style="width: 100%; align-items: center; max-height: 62px; text-align: center"
                  size="large"
                />
                <a-range-picker v-if="dateFilterType === 'range'" 
                v-model:value="selectedRangeDates" 
                format="D/M/YY" 
                style="width: 100%; align-items: center; max-height: 62px; text-align: center" 
                size="large"/>
                <a-radio-group class="mt-2" v-model:value="dateFilterType">
                  <a-radio-button value="span">Timespan</a-radio-button>
                  <a-radio-button value="single">Single</a-radio-button>
                  <a-radio-button value="range">Range</a-radio-button>
                </a-radio-group>
                <!-- <span class="d-flex justify-content-center fw-bolder text-dark me-3">{{ formattedDayOfWeek() }}</span> -->
              </div>
              <div class="flex-auto" style="width: 15rem">
                <a-select
                  v-model:value="selectedDepartments"
                  mode="multiple"
                  style="width: 100%; align-items: center; max-height: 62px;"
                  placeholder="Departments"
                  max-tag-count="responsive"
                  :options="$resources.departments"
                  :fieldNames="{label:'department', value: 'department'}"
                  size="large"
                >
                </a-select>
              </div>
              <div class="flex-auto" style="width: 15rem">
                <a-input v-model:value="searchValue" placeholder="Search" size="large">
                  <template #prefix>
                    <v-icon icon="mdi mdi-magnify" color="grey"></v-icon>
                  </template>
                </a-input>
              </div>
            </div>
          </div>
          
          <!-- Toolbar Actions -->
          <v-toolbar color="blue-lighten-5">
            <v-btn icon="mdi mdi-plus" @click="appointmentDialog('New Appointment', true)" rounded="0"></v-btn>
          </v-toolbar>

          <!-- Appointment Tab -->
          <v-tabs v-model="tab" align-tabs="center" color="indigo" bg-color="white" show-arrows>
            <v-tab v-for="(value, key) in groupedAppointments" :key="key" :value="key">
              {{ key }}
              <v-badge color="indigo" :content="getBadgeNumber(key)" inline></v-badge>
            </v-tab>
          </v-tabs>
          <div class="tab-content">
            <v-window v-model="tab" disabled>
              <v-window-item v-for="(value, key) in groupedAppointments" :key="key" :value="key">
                <appointmenttab 
                  :searchValue="searchValue"
                  :selectedDate="selectedDate"
                  :selectedDepartments="selectedDepartments" 
                  :appointments="value" 
                  :tab="key.toLowerCase()"
                  :loading="appointmentsLoading"
                  ref="appointmentTabRef"
                />
              </v-window-item>
            </v-window>
          </div>
          <!-- /Appointment Tab -->

        </div>
      </div>
    </div>
    <!-- /Page Content -->

    <!-- Page Dialogs -->
    <!-- / Page Dialogs -->
    <v-overlay
      :model-value="lodingOverlay"
      class="align-center justify-center"
    >
      <v-progress-circular
        color="primary"
        size="64"
        indeterminate
      ></v-progress-circular>
    </v-overlay>
  </div>
</template>

<script >
import { ref } from 'vue';
import moment from "moment";
import dayjs from 'dayjs';
import updateLocale from 'dayjs/plugin/updateLocale';
import isoWeek from 'dayjs/plugin/isoWeek';

dayjs.extend(updateLocale);
dayjs.extend(isoWeek);

// Update locale to set the start of the week to Sunday
dayjs.updateLocale('en', {
  weekStart: 0 // Start week on Sunday
});


import {VBadge} from 'vuetify/components/VBadge';
import {VTab, VTabs} from 'vuetify/components/VTabs';
import {VWindow, VWindowItem} from 'vuetify/components/VWindow';
import {VIcon} from 'vuetify/components/VIcon';
import {VToolbar, VToolbarItems} from 'vuetify/components/VToolbar';
import {VBtn} from 'vuetify/components/VBtn';
import { VDialog } from 'vuetify/components/VDialog';
import { VCard, VCardTitle, VCardText, VCardActions } from 'vuetify/components/VCard';
import { VAlert } from 'vuetify/components/VAlert';
import { VOverlay } from 'vuetify/components/VOverlay';
import { VProgressCircular } from 'vuetify/components/VProgressCircular';

export default {
  inject: ['$socket', '$call'],
  components: {
    VBadge, VTabs, VTab, VWindow, VWindowItem, VIcon, VToolbar, VToolbarItems, VBtn,
    VDialog, VCard, VCardTitle, VCardText, VCardActions, VAlert, VOverlay, VProgressCircular, 
  },
  data() {
    return {
      tab: null,
      dateFilterType: 'span',

      appointments: [],
      groupedAppointments: {Scheduled:[], 'Partially Completed':[],Completed:[], Complaint:[], 'No Show':[], Cancelled:[],},
      selectedSpan: 'today',
      selectedDate: ref(dayjs()),
      selectedRangeDates: [dayjs().startOf('isoWeek').subtract(1, 'day'), dayjs().endOf('isoWeek').subtract(1, 'day')],

      spans: ref([
        {label: 'Yesterday', value: 'yesterday'},
        {label: 'Today', value: 'today'},
        {label: 'Tomorrow', value: 'tomorrow'},

        {label: 'Last Week', value: 'last week'},
        {label: 'This Week', value: 'this week'},
        {label: 'Next Week', value: 'next week'},
        {label: 'Last Month', value: 'last month'},
        {label: 'This Month', value: 'this month'},
        {label: 'Next Month',  value: 'next month'},
      ]),


      searchValue: '',
      selectedDepartments: undefined,
      appointmentsLoading: true,
      appointmentOpen: false,
      vitalSignsOpen: false,
      serviceUnitOpen: false,
      transferOpen: false,
      paymentTypeOpen: false,
      lodingOverlay: false,
      slots: {},
      message: '',
      alertVisible: false,
      dateWeek: ref(dayjs()),
      appointmentForm: {},
    };
  },
  created() {
    this.fetchRecords();
    this.$socket.on('patient_appointments', response => {
      if(response){
        this.appointments = this.adjustAppointments(response)
        this.groupAppointmentsByStatus();
      }
    })
  },
  mounted() {
  },
  beforeUnmount() {
    // Clear the timeout before unmounting the component
    clearTimeout(this.timer);
  },
  methods: {
    showAlert(message, duration) {
      this.message = message;
      this.alertVisible = true;
      setTimeout(() => {
        this.alertVisible = false;
      }, duration);
    },
    fetchRecords() {
      this.appointmentsLoading = true;
      this.$call('pest.api.methods.fetch_patient_appointments')
      .then(response => {
        this.appointments = this.adjustAppointments(response)
        this.groupAppointmentsByStatus();
        this.appointmentsLoading = false;
      })
      .catch(error => {
        this.appointmentsLoading = false;
        console.error('Error fetching records:', error);
      });
    },
    adjustAppointments(data) {
			return [...(data || [])].map((d) => {
        try {
          if(typeof d.patient_details === 'string'){
            d.patient_details = JSON.parse(d.patient_details)
            d.visit_notes = JSON.parse(d.visit_notes)
            d.status_log = JSON.parse(d.status_log)
            d.arriveTime = '-'
            d.status_log.forEach(value => {
              if(value.status == 'Arrived')
                d.arriveTime = dayjs(value.time)
            })
          }
        } catch (error) {
          console.error('Error parsing JSON:', error);
        }

				d.appointment_time_moment = dayjs(d.appointment_date + ' ' + d.appointment_time).format('h:mm a');
				d.patient_cpr = d.patient_name + ' ' + d.patient_details.cpr

				return d;
			});
		},
    getBadgeNumber(tab){
      let count = 0
      if(this.groupedAppointments[tab]){
        count = this.groupedAppointments[tab].reduce((total, value) => {
          let departmentFilter = true
          if(this.selectedDepartments)
            departmentFilter = this.selectedDepartments.some(department => {return department === value.department})
          if(this.selectedDate.format('YYYY-MM-DD') === value.appointment_date && departmentFilter){
            return total + 1
          }
          return total
        },0)
      }
      return count
    },
    groupAppointmentsByStatus() {
      this.groupedAppointments = {Scheduled:[], 'Partially Completed':[],Completed:[], Complaint:[], 'No Show':[], Cancelled:[],}
      this.appointments.forEach(appointment => {
        const status = appointment.visit_status;
        if (!this.groupedAppointments[status])
          this.groupedAppointments[status] = [];
        this.groupedAppointments[status].push(appointment);
      });
    },
    spanToDate(span) {
      if(span){
        if(span === 'yesterday')
          return dayjs().add(-1, 'd')
        if(span === 'today')
          return dayjs()
        if(span === 'tomorrow')
          return dayjs().add(1, 'd')
        if(span === 'last week')
          return [dayjs().subtract(1, 'week').startOf('isoWeek').subtract(1, 'day'), dayjs().subtract(1, 'week').endOf('isoWeek').subtract(1, 'day')]
        if(span === 'this week')
          return [dayjs().startOf('isoWeek').subtract(1, 'day'), dayjs().endOf('isoWeek').subtract(1, 'day')]
        if(span === 'next week')
          return [dayjs().add(1, 'week').startOf('isoWeek').subtract(1, 'day'), dayjs().add(1, 'week').endOf('isoWeek').subtract(1, 'day')]
        if(span === 'last month')
          return [dayjs().subtract(1, 'month').startOf('month'), dayjs().subtract(1, 'month').endOf('month')]
        if(span === 'this month')
          return [dayjs().startOf('month'), dayjs().endOf('month')]
        if(span === 'next month')
          return [dayjs().add(1, 'month').startOf('month'), dayjs().add(1, 'month').endOf('month')]
        return undefined
      }
      return undefined
		},
    vitalSignDialog(row) {
      this.appointmentForm.name = row.appointment_id;
      this.appointmentForm.patient = row.patient_details.id;
			this.vitalSignsOpen = true;
		},
    transferPractitionerDialog(row) {
      this.appointmentForm.name = row.appointment_id;
      this.appointmentForm.practitioner = row.practitioner;
      this.appointmentForm.practitioner_name = row.practitioner_name;
			this.transferOpen = true
		},
    serviceUnitDialog(row) {
      this.appointmentForm.name = row.appointment_id;
      this.appointmentForm.service_unit = row.service_unit;
			this.serviceUnitOpen = true
		},
    paymentTypeDialog(row) {
      this.appointmentForm.name = row.appointment_id;
      this.appointmentForm.custom_payment_type = row.payment_type;
			this.paymentTypeOpen = true
    },
    customWeekStartEndFormat: value => `${dayjs(value).startOf('week').format('MMMM D, YYYY')} -> ${dayjs(value).endOf('week').format('MMMM D, YYYY')}`,
    showSlots() {
      if (this.appointmentForm.appointment_date && this.appointmentForm.practitioner) {
        this.slots = {};
        let tempForm = this.appointmentForm;
        tempForm.__islocal = 1;
        tempForm.__unsaved = 1;
        this.$call('healthcare.healthcare.doctype.patient_appointment.patient_appointment.get_availability_data', {
          practitioner: this.appointmentForm.practitioner,
          date: this.appointmentForm.appointment_date.format('YYYY/MM/DD'),
          appointment: JSON.stringify(tempForm)
        }).then((data) => {
          if (data.slot_details.length > 0) {
            // make buttons for each slot
            this.getSlots(data);
          } else {
            let message = error.message.split('\n');
            message = message.find(line => line.includes('frappe.exceptions'));
            const firstSpaceIndex = message.indexOf(' ');
            this.showAlert(message.substring(firstSpaceIndex + 1) , 10000)
          }
        }).catch(error => {
          let message = error.message.split('\n');
          message = message.find(line => line.includes('frappe.exceptions'));
          const firstSpaceIndex = message.indexOf(' ');
          this.showAlert(message.substring(firstSpaceIndex + 1) , 10000)
        });
      } 
	  },
    getSlots(data) {
      let appointment_count = 0;
      let slot_start_time, slot_end_time, available_slots;
      let appointment_date = this.appointmentForm.appointment_date

      data.slot_details.forEach((slot_info) => {
        slot_info.avail_slot.map(slot => {
          appointment_count = 0;
          slot.disabled = false;
          slot.count_class = '';
          slot_start_time = moment(slot.from_time, 'HH:mm:ss');
          slot_end_time = moment(slot.to_time, 'HH:mm:ss');
          slot.interval = (slot_end_time - slot_start_time) / 60000 | 0;

          // restrict past slots based on the current time.
          let now = moment();
          let booked_moment = ""
          if((now.format("YYYY-MM-DD") == appointment_date) && (slot_start_time.isBefore(now) && !slot.maximum_appointments)){
            slot.disabled = true;
          } else {
            // iterate in all booked appointments, update the start time and duration
            slot_info.appointments.forEach((booked) => {
              booked_moment = moment(booked.appointment_time, 'HH:mm:ss');
              let end_time = booked_moment.clone().add(booked.duration, 'minutes');

              // to get apointment count for all day appointments
              if (slot.maximum_appointments) {
                if (booked.appointment_date == appointment_date) {
                  appointment_count++;
                }
              }
              // Deal with 0 duration appointments
              if (booked_moment.isSame(slot_start_time) || booked_moment.isBetween(slot_start_time, slot_end_time)) {
                if (booked.duration == 0) {
                  slot.disabled = true;
                  return false;
                }
              }

              // Check for overlaps considering appointment duration
              if (slot_info.allow_overlap != 1) {
                if (slot_start_time.isBefore(end_time) && slot_end_time.isAfter(booked_moment)) {
                  // There is an overlap
                  slot.disabled = true;
                  return false;
                }
              } else {
                if (slot_start_time.isBefore(end_time) && slot_end_time.isAfter(booked_moment)) {
                  appointment_count++;
                }
                if (appointment_count >= slot_info.service_unit_capacity) {
                  // There is an overlap
                  slot.disabled = true;
                  return false;
                }
              }
            });
          }
          if (slot_info.allow_overlap == 1 && slot_info.service_unit_capacity > 1) {
            available_slots = slot_info.service_unit_capacity - appointment_count;
            slot.count = `${(available_slots > 0 ? available_slots : 'Full')}`;
            slot.count_class = `${(available_slots > 0 ? 'badge-success' : 'badge-danger')}`;
            slot.tool_tip =`${available_slots} slots available for booking`;
          }

          if (slot.maximum_appointments) {
            if (appointment_count >= slot.maximum_appointments) {
              slot.disabled = true;
            }
            else {
              slot.disabled = false;
            }
            available_slots = slot.maximum_appointments - appointment_count;
            slot.count = `${(available_slots > 0 ? available_slots : 'Full')}`;
            slot.count_class = `${(available_slots > 0 ? 'badge-success' : 'badge-danger')}`;
          } 
        });
      });
      this.slots = data;
    },
    onSubmitTransferPractitioner() {
      this.lodingOverlay = true;
      this.$call('pest.api.methods.transferToPractitioner', 
        {app: this.appointmentForm.name, practitioner: this.appointmentForm.practitioner}
      ).then(response => {
        this.lodingOverlay = false;
        this.transferOpen = false;
      }).catch(error => {
        console.error(error);
        let message = error.message.split('\n');
        message = message.find(line => line.includes('frappe.exceptions'));
        if(message){
          const firstSpaceIndex = message.indexOf(' ');
          this.showAlert(message.substring(firstSpaceIndex + 1) , 10000)
        }
      });
    },
    onSubmitServiceUnit() {
      this.lodingOverlay = true;
      this.$call('frappe.client.set_value', 
        {doctype: 'Patient Appointment', name: this.appointmentForm.name, fieldname: 'service_unit', value: this.appointmentForm.service_unit}
      ).then(response => {
        this.lodingOverlay = false;
        this.serviceUnitOpen = false;
      }).catch(error => {
        console.error(error);
        let message = error.message.split('\n');
        message = message.find(line => line.includes('frappe.exceptions'));
        if(message){
          const firstSpaceIndex = message.indexOf(' ');
          this.showAlert(message.substring(firstSpaceIndex + 1) , 10000)
        }
      });
    },
    onSubmitPaymentType() {
      this.lodingOverlay = true;
      this.$call('frappe.client.set_value', 
        {doctype: 'Patient Appointment', name: this.appointmentForm.name, fieldname: 'custom_payment_type', value: this.appointmentForm.custom_payment_type}
      ).then(response => {
        this.lodingOverlay = false;
        this.paymentTypeOpen = false;
      }).catch(error => {
        console.error(error);
        let message = error.message.split('\n');
        message = message.find(line => line.includes('frappe.exceptions'));
        if(message){
          const firstSpaceIndex = message.indexOf(' ');
          this.showAlert(message.substring(firstSpaceIndex + 1) , 10000)
        }
      });
    },
    createRange (length, start) {
      return Array.from({ length }).map((_, i) => i + start)
    },
    formattedDayOfWeek() {
      if (!this.selectedDate) return '';
      return dayjs(this.selectedDate).format('dddd');
    },
  },
  name: 'Appointments',
};
</script>

<style>
.appointment-tab .ant-picker-input input{
  cursor: 'pointer';
  text-align: center;
}
</style>