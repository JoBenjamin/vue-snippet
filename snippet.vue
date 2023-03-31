<template lang="pug">
template(v-slot:top)
  v-row.my-2(v-if="showFilters")
    v-col(lg="3" md="4" sm="6" v-for="(group, groupIndex) in filterGroupedIncidents")
      v-expansion-panels.my-n3(flat)
        v-expansion-panel
          v-expansion-panel-header
            .d-flex.align-center
              v-icon.mr-2(:color="getSelectedCount(groupIndex) > 0 ? 'primary' : ''") {{ group.icon }}
              .text-p {{ `${$t(`incidentGroups.${groupIndex}`)} ${getSelectedCount(groupIndex)}/${group.incidents.length}` }}
              v-spacer
              v-checkbox.mt-0.pt-0(
                hide-details
                :ripple="false"
                @click.stop="handleFilterCategoryClick(groupIndex, getSelectedCount(groupIndex) === group.incidents.length)"
                :input-value="getSelectedCount(groupIndex) === group.incidents.length"
              )
          v-expansion-panel-content.pt-2.mb-n3
            v-checkbox.my-n3(
              v-for="(incident, index) in group.incidents"
              :key="incident"
              :label="`${$t(`incidents.${incident}`)} ${getActiveIncidentCount(incident) ? `(${getActiveIncidentCount(incident)} db) ` : ''}`"
              v-model="visibleIncidents[incident]"
            )
</template>

<script>
  export default {
    data() {
      return {
        tvmmachines: [],
        incidentMachines: null,
        visibleIncidents: {},
        showFilters: false,
      };
    },
    computed: {
      filterGroupedIncidents() {
        const groupedList = this.groupIncidents(
          incidentList.map((incident) => incident.value),
          true,
        );
        return groupedList;
      },
      filteredIncidentMachines() {
        if (!this.incidentMachines) {
          return [];
        }
        const clone = _.cloneDeep(this.incidentMachines);
        this.applyIncidentFilters(clone);
        return clone.filter((tvm) => tvm.incidents.length);
      },
    },
    methods: {
      groupIncidents(incidents, filterMode = false) {
        const result = {};
        incidents.forEach((incident) => {
          const incidentGroup = this.findIncidentGroup(incident, filterMode);
          if (!Object.hasOwn(result, incidentGroup.value)) {
            result[incidentGroup.value] = {
              incidents: [],
              icon: incidentGroup.icon,
            };
          }
          result[incidentGroup.value].incidents.push(incident);
        });
        return result;
      },
      handleFilterCategoryClick(group, allSelected) {
        const incidents = this.getIncidentsByFilterGroup(group);
        incidents.forEach((incident) => {
          this.$set(this.visibleIncidents, incident.value, !allSelected);
        });
      },
      getActiveIncidents(tvmList = []) {
        const incidents = [];
        tvmList.forEach((tvm) => {
          tvm.incidents.forEach((incident) => {
            if (incidents.includes(incident)) return;
            const newIncident = this.getIncidentObject(incident);
            if (!newIncident) {
              console.warn('No incident found for', incident);
              return;
            }
            incidents.push(newIncident);
          });
        });
        return incidents;
      },

      getSelectedCount(filterGroup) {
        let selectedCount = 0;
        const entries = Object.entries(this.visibleIncidents);
        entries.forEach((entry) => {
          const [incident, value] = entry;
          if (!value) return;
          const incidentObject = this.getIncidentObject(incident);
          if (incidentObject && incidentObject.filterGroup.value === filterGroup) selectedCount++;
        });
        return selectedCount;
      },
      getActiveIncidentCount(incident) {
        if (!this.incidentMachines) return 0;
        const activeIncidents = this.incidentMachines.flatMap((tvm) => tvm.incidents);
        const hits = activeIncidents.filter((activeIncident) => activeIncident === incident);
        return hits.length;
      },
      applyIncidentFilters(tvms) {
        tvms.forEach((tvm) => {
          const filteredIncidents = this.filterIncidents(tvm.incidents);
          tvm.incidents = filteredIncidents;
        });
      },
      filterIncidents(incidents) {
        const replica = [...incidents];
        const prefilteredIncidents = this.prefilterIncidents(replica);
        const visibleResult = prefilteredIncidents.filter(this.isIncidentVisible);
        return visibleResult;
      },
      prefilterIncidents(incidents) {
        if (incidents.includes('POWER_ERROR')) {
          return ['POWER_ERROR'];
        }
        return incidents;
      },
      isIncidentVisible(incident) {
        return this.visibleIncidents[incident];
      },
    },
  };
</script>
