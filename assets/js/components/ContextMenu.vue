<template>
  <vue-context
    ref="menu"
    :close-on-click="false"
    :close-on-scroll="false"
  >
    <li class="header">
      {{ selectedIssues.length }}
      <template v-if="selectedIssues.length <= 1">
        {{ $t('numéro sélectionné') }}
      </template>
      <template v-else>
        {{ $t('numéros sélectionnés') }}
      </template>
    </li>
    <li class="menu-separator">
      {{ $t('Etat') }}
    </li>
    <li
      v-for="(text, id) in conditionStates"
      :key="`condition-${id}`"
      :class="{item: true, selected: condition === id, 'issue-condition': true, [`issue-condition-${id}`]: true}"
      @click="condition = id"
    >
      {{ text }}
    </li>
    <li class="menu-separator">
      {{ $t('Date d\'achat') }}
    </li>
    <li
      v-for="(text, id) in purchaseStates"
      :key="`purchase-${id}`"
      :class="{item: true, selected: currentPurchaseId === id, 'purchase-state': true, 'v-context__sub': id === 'link', [id]: true }"
      @click="currentPurchaseId = id"
    >
      <BIconCalendar v-if="id === 'link'" />
      <BIconCalendarX v-if="id === 'unlink'" />
      {{ text }}
      <ul
        v-if="id === 'link'"
        class="v-context"
      >
        <li class="menu-separator">
          <h5
            v-if="!newPurchaseContext"
            @click="newPurchaseContext = !newPurchaseContext"
          >
            {{ $t('Nouvelle date d\'achat...') }}
          </h5>
          <template v-else>
            <input
              v-model="newPurchaseDescription"
              required
              type="text"
              class="form-control"
              size="30"
              maxlength="30"
              :placeholder="$t('Description')"
            >
            <input
              v-model="newPurchaseDate"
              required
              type="date"
              class="form-control"
              size="30"
              maxlength="10"
              :placeholder="$t('Date d\'achat')"
              @keydown.prevent="() => {}"
            >
            <button
              class="btn btn-default"
              @click="createPurchaseDate()
                      newPurchaseDescription = newPurchaseDate = null
                      newPurchaseContext = false"
            >
              {{ $t('Créer') }}
            </button>
            <button
              class="btn btn-default cancel"
              @click="newPurchaseContext = false"
            >
              {{ $t('Annuler') }}
            </button>
          </template>
        </li>
        <li
          v-for="{id: purchaseId, date, description} in purchases"
          :key="purchaseId"
          :class="{item: true, selected: currentPurchaseId === purchaseId, 'purchase-date': true}"
          class="item purchase-date"
          @click.stop="currentPurchaseId = purchaseId"
        >
          <b>{{ description }}</b><br>{{ date }}
        </li>
      </ul>
    </li>
    <li
      class="footer"
      @click="updateSelectedIssues"
    >
      {{ $t('Enregistrer les changements') }}
    </li>
  </vue-context>
</template>

<script>
import l10nMixin from "../mixins/l10nMixin";
import VueContext from "vue-context";
import conditionMixin from "../mixins/conditionMixin";
import { BIconCalendar, BIconCalendarX, BIconTag } from "bootstrap-vue";

export default {
  name: "ContextMenu",
  components: {
    VueContext,
    BIconCalendar,
    BIconCalendarX
  },
  mixins: [l10nMixin, conditionMixin],
  props: {
    publicationCode: {
      type: String, required: true
    },
    selectedIssues: {
      type: Array, required: true
    },
    purchases: {
      type: Array, required: true
    },
  },
  emits: ['update-issues', 'create-purchase'],
  data: () => ({
    condition: 'do_not_change',
    isToSell: 'do_not_change',
    currentPurchaseId: 'do_not_change',
    newPurchaseContext: false,
    newPurchaseDescription: '',
    newPurchaseDate: '',
  }),

  computed: {
    conditionStates() {
      return {
        do_not_change: this.$t("Conserver l'état actuel"),
        missing: this.$t("Marquer comme non-possédé(s)"),
        possessed: this.$t("Marquer comme possédé(s)"),
        bad: this.$t("Marquer comme en mauvais état"),
        notsogood: this.$t("Marquer comme en état moyen"),
        good: this.$t("Marquer comme en bon état")
      }
    },
    purchaseStates() {
      return {
        do_not_change: this.$t("Conserver la date d'achat"),
        link: this.$t("Associer avec une date d'achat"),
        unlink: this.$t("Désassocier de la date d'achat")
      }
    }
  },

  methods: {
    async updateSelectedIssues() {
      const vm = this
      this.$emit('update-issues', {
        publicationCode: this.publicationCode,
        issueNumbers: this.selectedIssues,
        condition: (vm.conditions.find(({value}) => value === vm.condition) || {dbValue: null}).dbValue,
        istosell: this.isToSell,
        purchaseId: this.currentPurchaseId
      })
    },
    async createPurchaseDate() {
      this.$emit('create-purchase', {
        date: this.newPurchaseDate,
        description: this.newPurchaseDescription,
      })
    }
  }
}
</script>

<style scoped lang="scss">
.v-context {
  padding: 0;

  li {
    padding: 0 30px;

    &.item:hover, &.footer:hover {
      background-color: #4f5b69 !important;
      color: #fff;
      cursor: pointer;
    }

    &.header, &.footer {
      height: 30px;
      line-height: 25px;
      font-size: 14px;
      background-color: #3d4b5f;
      color: white;
      padding: 2px;
      font-weight: bold;
      text-align: center;

      &.footer {
        background-color: green;
        border-top: 3px solid #e6e6e6;
      }
    }

    &.menu-separator {
      color: black;
      font-size: 12px;
      font-weight: bold;
      line-height: 20px;
      text-align: center;
      border: 1px solid #e6e6e6;
      border-top-width: 3px;
      padding: .2em;
    }

    &.item {
      display: flex;
      align-items: center;
      color: black;
      line-height: 25px;
      background-position: left center;

      &.selected {
        background-color: #a52a2a;
        color: white;
      }

      &.purchase-state {
        &.link {
          &:after {
            position: absolute;
            font-weight: bold;
            right: 10px;
            content: '>'
          }
        }
      }

      &.purchase-date {
        flex-direction: column;
        background-repeat: no-repeat;
        background-size: 12px;
        background-position-x: 10px;
        line-height: 15px;
      }

      &.issue-condition:before {
        position: absolute;
        left: 0;
        content: " ";
        width: 0;
        height: 0;
        border-radius: 50%;
        margin: 6px;
      }

      &.issue-condition-missing:before {
        border: 8px solid black;
      }

      &.issue-condition-bad:before {
        border: 8px solid red;
      }

      &.issue-condition-notsogood:before {
        border: 8px solid orange;
      }

      &.issue-condition-good:before {
        border: 8px solid #2CA77B;
      }

      &.issue-condition-possessed:before {
        border: 8px solid #808080;
      }

      &.v-context__sub {
        * {
          color: black;
        }

        .item {
          line-height: 15px;
          padding-top: 10px;

          &:hover {
            color: white;
          }
        }
      }

      svg {
        margin-left: -24px;
        margin-right: 8px;
        font-size: 16px;
      }
    }
  }
}
</style>
