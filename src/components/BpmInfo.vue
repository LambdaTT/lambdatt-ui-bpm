<template>
  <div v-if="showTitle" class="col-12 q-px-md q-py-sm q-mx-sm bg-teal-6 container title">
    <i class="fa-solid fa-list-check"></i>
    <span>Andamento</span>
  </div>
  <div class="row q-pa-sm q-mx-sm">
    <div :class="`col-12 ${wrapFields ? '' : 'col-md-6'} q-px-sm q-mb-md`">
      <div class="container">
        <h1><i v-if="!hideIcons" class="fas fa-traffic-light section-icon"></i></h1>
        <h1 v-if="!hideLabels">Situação:</h1>
      </div>

      <div class="container item">
        <i v-if="!hideIcons" class="fa-solid fa-caret-right item-icon"></i>
        <span>{{ data.stepName }}</span>
      </div>
    </div>

    <div :class="`col-12 ${wrapFields ? '' : 'col-md-6'} q-px-sm q-mb-md`">
      <div class="container">
        <h1><i v-if="!hideIcons" class="fas fa-clock-rotate-left section-icon"></i></h1>
        <h1 v-if="!hideLabels">Histórico:</h1>
      </div>

      <div class="container item">
        <ul>
          <li v-for="(step, index) in data.stepHistory" :key="index">
            <i v-if="!hideIcons" class="fa-solid fa-caret-right item-icon"></i>{{ step.date }} - {{ step.label }}
          </li>
        </ul>
      </div>
    </div>

  </div>
</template>

<script>
import ENDPOINTS from '../ENDPOINTS.js'

// Components:
export default {
  name: 'ui-bpm-bpminfo',

  props: {
    hideIcons: Boolean,
    hideLabels: Boolean,
    showTitle: Boolean,
    wrapFields: Boolean,
    preTransitionFn: Function,
    postTransitionFn: Function,
    modelValue: {
      type: Object,
      default: () => ({}), // Garante um objeto vazio como padrão
      required: true
    },
    ExecutionKey: {
      type: String,
      default: () => null,
    }
  },

  data() {
    return {
      data: {
        stepName: null,
        availableActions: [],
        stepHistory: [],
      },
    }
  },

  computed: {
    factory() {
      return {
        data: { ...this.data },
      }
    }
  },

  methods: {
    async getExecutionInfo() {
      try {
        const response = await this.$getService('toolcase/http').get(`${ENDPOINTS.EXECUTION_INFO}/${this.ExecutionKey}`);
        // Fills Data
        for (let key in this.data) {
          if (key in response.data) {
            this.data[key] = response.data[key];
          }
        }
      } catch (error) {
        console.error("An error occurred while attempting to retrieve the execution's data.", error);
      }
    },

    async getTransitions() {
      try {
        const response = await this.$getService('toolcase/http').get(`${ENDPOINTS.AVAILABLE_TRANSITIONS}/${this.ExecutionKey}`);
        this.data.availableActions = response.data.map(action => ({
          icon: action.ds_icon,
          label: action.ds_title,
          value: action.id_bpm_transition,
          fn: async () => {
            try {
              // Confimation
              if (!confirm(`Deseja proceder com a ação: ${action.ds_title}?`)) { return false; }

              this.$getService('toolcase/loader').load('bpm-transition-execute');

              if (this.preTransitionFn) {
                await this.preTransitionFn()
              }

              await this.executeTransition(action.ds_key);
              await this.refresh();

              if (this.postTransitionFn) {
                await this.postTransitionFn()
              }

              this.$getService('toolcase/utils').notify({
                message: 'Ação realizada com sucesso',
                type: 'positive',
                position: 'top-right'
              });
            } catch (error) {
              throw error;
            } finally {
              this.$getService('toolcase/loader').loaded('bpm-transition-execute');
            }
          },
        }))
      } catch (error) {
        console.error("An error occurred while attempting to retrieve the transitions' data.", error);
      }
    },

    async getStepHistory() {
      try {
        const response = await this.$getService('toolcase/http').get(`${ENDPOINTS.STEP_TRACKING}/${this.ExecutionKey}`);
        this.data.stepHistory = response.data.map(step => ({
          date: step.dtTracking,
          label: step.stepName,
        }))
      } catch (error) {
        console.error("An error occurred while attempting to retrieve the professionals' data.", error);
      }
    },

    async executeTransition(transitionKey) {
      try {
        await this.$getService('toolcase/http').put(`${ENDPOINTS.TRANSITION}/${this.ExecutionKey}/${transitionKey}`);
      } catch (error) {
        console.error(`Failed to execute transition ${transitionKey}.`, error);
      }
    },

    async refresh() {
      await this.getExecutionInfo()
      await this.getTransitions()
      await this.getStepHistory()
    },
  },

  watch: {
    // Sincroniza alterações de bpm para o pai
    data: {
      handler() {
        this.$emit("update:model-value", this.factory);
      },
      deep: true,
    },

    async ExecutionKey(val) {
      if (!!val)
        await this.refresh()
    }
  },

  mounted() {
    this.$emit("update:model-value", this.factory);
  }
}
</script>

<style scoped>
h1 {
  font-size: 1.5rem;
  line-height: 20px;
}

i {
  margin-right: 10px;
  width: 25px;
  display: flex;
  justify-content: center;
}

.row {
  background-color: rgba(0, 0, 0, 0.05);
  border: 1px solid rgba(0, 0, 0, 0.05);
  /* Largura, estilo, cor */
}

.title {
  color: white;
  font-size: 1.5rem;
  font-weight: normal;
  border-top-left-radius: 5px;
  border-top-right-radius: 5px;
}

.container {
  display: flex;
  flex-direction: row;
  align-items: center;
}

.item {
  font-size: 1rem;
}

.item ul,
.item li {
  margin: 0;
  padding: 0;
}

.item li {
  display: flex;
  align-items: baseline;
  list-style-type: none;
}
</style>