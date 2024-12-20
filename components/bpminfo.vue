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
        <span>{{ bpm.data.stepName }}</span>
      </div>
    </div>

    <div :class="`col-12 ${wrapFields ? '' : 'col-md-6'} q-px-sm q-mb-md`">
      <div class="container">
        <h1><i v-if="!hideIcons" class="fas fa-clock-rotate-left section-icon"></i></h1>
        <h1 v-if="!hideLabels">Histórico:</h1>
      </div>

      <div class="container item">
        <ul>
          <li v-for="(step, index) in bpm.data.stepHistory" :key="index">
            <i v-if="!hideIcons" class="fa-solid fa-caret-right item-icon"></i>{{ step.date }} - {{ step.label }}
          </li>
        </ul>
      </div>
    </div>

  </div>
</template>

<script>
// Services:
import { ENDPOINTS } from 'src/services/endpoints'

// Components:
export default {
  name: 'components-common-bpminfo',

  components: {
  },

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
    },
  },

  data() {
    return {
      bpm: {
        data: {
          stepName: null,
          executionKey: null,
          id_bpm_execution: null,
          availableActions: [],
          stepHistory: [],
        },
        function: {
          read: this.read,
        },
      },
    }
  },

  computed: {
  },

  methods: {
    async read(source) {
      // Fills Data
      for (let key in this.bpm.data) {
        if (key in source) {
          this.bpm.data[key] = source[key];
        }
      }
      // Fills Available Transitions
      if (source.executionKey) { await this.getTransitions(source.executionKey); }

      // Fills Step History
      if (source.id_bpm_execution) { await this.getStepHistory(source.id_bpm_execution); }
    },

    async getTransitions(executionKey) {
      try {
        const response = await this.$http.get(`${ENDPOINTS.BPM.AVAILABLE_TRANSITIONS}/${executionKey}`);
        this.bpm.data.availableActions = response.data.map(action => ({
          icon: action.ds_icon,
          label: action.ds_title,
          value: action.id_bpm_transition,
          fn: async () => {
            try {
              // Confimation
              if (!confirm(`Deseja proceder com a ação: ${action.ds_title}?`)) { return false; }

              if (this.preTransitionFn) {
                await this.preTransitionFn()
              }

              await this.executeTransition(executionKey, action.ds_key);

              if (this.postTransitionFn) {
                await this.postTransitionFn()
              }
            } catch (error) {
              throw error;
            }
          },
        }))
      } catch (error) {
        console.error("An error occurred while attempting to retrieve the transitions' data.", error);
      }
    },

    async getStepHistory(executionId) {
      try {
        const response = await this.$http.get(`${ENDPOINTS.BPM.STEP_TRACKING}/${executionId}`);
        this.bpm.data.stepHistory = response.data.map(step => ({
          date: step.dtTracking,
          label: step.stepName,
        }))
      } catch (error) {
        console.error("An error occurred while attempting to retrieve the professionals' data.", error);
      }
    },

    async executeTransition(executionKey, transitionKey) {
      try {
        const response = await this.$http.put(`${ENDPOINTS.BPM.DO_TRANSITION}/${executionKey}/${transitionKey}`);
      } catch (error) {
        console.error(`Failed to execute transition ${transitionKey}.`, error);
      }
    }
  },

  watch: {
    // Sincroniza alterações de bpm para o pai
    bpm: {
      handler(newValue) {
        this.$emit("update:model-value", newValue);
      },
      deep: true,
    },

    // Sincroniza alterações do pai para o bpm
    modelValue: {
      handler(newValue) {
        if (newValue) {
          Object.assign(this.bpm.data, newValue.data); // Copia os valores do pai para o filho
        }
      },
      deep: true,
    },
  },

  mounted() {
    // Sincroniza bpm inicial com o pai
    if (Object.keys(this.modelValue).length === 0) {
      this.$emit("update:model-value", this.bpm);
    } else {
      Object.assign(this.bpm.data, this.modelValue.data);
    }
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