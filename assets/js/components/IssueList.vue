<template>
  <div v-if="publicationName">
    <Publication
      size="xl"
      :publicationcode="publicationcode"
      :publicationname="publicationName"
    />
    <div v-if="issues">
      <div
        v-once
        class="issue-filter"
      >
        <table>
          <tr
            v-for="conditionFilter in ['possessed', 'missing']"
            :key="conditionFilter"
          >
            <td>
              <input
                :id="`show-${conditionFilter}`"
                v-model="filter[conditionFilter]"
                type="checkbox"
              >
            </td>
            <td>
              <label
                :for="`show-${conditionFilter}`"
              >
                <template v-if="conditionFilter === 'possessed'">{{ $t('Afficher les numéros possédés') }}</template>
                <template v-else-if="conditionFilter === 'missing'">{{ $t('Afficher les numéros manquants') }}</template>
                ({{
                  issues.filter(issue => conditionFilter === "possessed" ? issue.condition : !issue.condition).length
                }})
              </label>
            </td>
          </tr>
        </table>
      </div>
      <div
        class="issue-list"
        @contextmenu.prevent="openContextMenuIfBookNotOpen"
      >
        <b-alert
          v-if="userIssuesNotFoundForPublication.length"
          show
          variant="warning"
        >
          {{ $t('Certains des numéros que vous possédez pour ce magazine n\'existent plus. Cela peut se produire lorsque des numéros ont été renommés. Pour chaque numéro n\'existant plus, trouvez le numéro de remplacement, puis supprimez l\'ancien numéro en cliquant sur le bouton correspondant ci-dessous.') }}
          <ul>
            <li
              v-for="issueNotFound in userIssuesNotFoundForPublication"
              :key="`notfound-${issueNotFound}`"
            >
              {{ $t('n°') }}{{ issueNotFound.issueNumber }} <b-btn
                size="sm"
                @click="deletePublicationIssues([issueNotFound])"
              >
                {{ $t('Supprimer') }}
              </b-btn>
            </li>
          </ul>
        </b-alert>
        <b-alert
          v-once
          show
          variant="info"
          class="mb-0"
        >
          {{ $t('Cliquez sur les numéros que vous souhaitez ajouter à votre collection,') }}
          <span v-if="isTouchScreen">{{ $t('puis faites un appui long pour indiquer son état et validez.') }}</span>
          <span v-else>{{ $t('puis faites un clic droit pour indiquer son état et validez.') }}</span>
        </b-alert>
        <Book
          v-if="currentIssueOpened"
          :publication-code="currentIssueOpened.publicationcode"
          :issue-number="currentIssueOpened.issueNumber"
          @close-book="currentIssueOpened = null"
        />
        <div
          v-for="({issueNumber, title, condition, purchaseId}, i) in filteredIssues"
          :key="issueNumber"
          :class="{
            issue: true,
            [`issue-${condition ? 'possessed' : 'missing'}`]: true,
            preselected: preselected.includes(issueNumber),
            selected: selected.includes(issueNumber)
          }"
          :name="issueNumber"
          @mousedown.self.left="preselectedIndexStart = preselectedIndexEnd = i"
          @mouseup.self.left="updateSelected"
          @mouseover="preselectedIndexEnd = preselectedIndexStart === null ? null : i"
        >
          <a :name="issueNumber" />
          <b-icon-eye-fill
            :id="`issue-details-${issueNumber}`"
            class="preview mx-2"
            :alt="$t('Voir')"
            @mouseover="hoveredIssueNumber=issueNumber"
            @mouseout="hoveredIssueNumber=null"
            @click.prevent="currentIssueOpened = {publicationcode, issueNumber}"
          />
          <span
            v-once
            class="issue-text"
          >
            {{ $t('n°') }}{{ issueNumber }}
            <span class="issue-title">{{ title }}</span>
          </span>
          <div
            class="issue-details-wrapper"
          >
            <div class="issue-details">
              <input
                type="checkbox"
                disabled
                :checked="selected.includes(issueNumber)"
                @click.prevent="false"
              >
            </div>
            <Condition
              v-if="condition"
              :publicationcode="publicationcode"
              :issuenumber="issueNumber"
              :value="condition"
              :class="{
                'issue-details': true,
                'issue-condition': true
              }"
            />
            <div
              v-if="purchaseId && purchases && purchases.find(({id}) => id === purchaseId)"
              v-once
              class="issue-details issue-date"
            >
              <BIconCalendar :title="`${$t('Acheté le')} ${purchases.find(({id}) => id === purchaseId).date}`" />
            </div>
            <div
              v-else
              class="issue-details"
            />
          </div>
        </div>
        <IssueDetailsPopover
          v-if="hoveredIssueNumber"
          :publication-code="publicationcode"
          :issue-number="hoveredIssueNumber"
          placement="right"
        />
      </div>
      <ContextMenu
        v-if="purchases"
        ref="contextMenu"
        :publication-code="publicationcode"
        :selected-issues="selected"
        :purchases="purchases"
        @update-issues="updateIssues"
        @create-purchase="createPurchase"
      />
    </div>
    <div v-else-if="loading">
      {{ $t('Chargement...') }}
    </div>
  </div>
  <div v-else-if="!publicationNameLoading">
    <b-alert
      variant="danger"
      show
    >
      <div class="mb-4">
        {{ $t('Aucun numéro n\'est répertorié pour') }} {{ publicationcode.split("/")[1] }} ({{ $t('Pays de publication') }} : {{
          country
        }})
      </div>
      <div v-if="userIssuesForPublication.length">
        {{ $t('Souhaitez-vous supprimer ce magazine de votre collection ? Les numéros suivants seront supprimés de votre collection dans ce cas :') }}
        <ul>
          <li
            v-for="issue in userIssuesForPublication"
            :key="issue.issueNumber"
          >
            {{ issue.issueNumber }}
          </li>
        </ul>
        <b-btn
          variant="danger"
          @click="deletePublicationIssues(userIssuesForPublication)"
        >
          {{ $t('Supprimer') }}
        </b-btn>
      </div>
    </b-alert>
  </div>
</template>

<script>
import l10nMixin from "../mixins/l10nMixin";
import { mapActions, mapState } from "vuex";
import ContextMenu from "./ContextMenu";
import "vue-context/src/sass/vue-context.scss";
import axios from "axios";
import conditionMixin from "../mixins/conditionMixin";
import collectionMixin from "../mixins/collectionMixin";
import IssueDetailsPopover from "./IssueDetailsPopover";
import Book from "./Book";
import Condition from "./Condition";
import { BIconCalendar } from "bootstrap-vue";
import Publication from "./Publication";

export default {
  name: "IssueList",
  components: {
    Publication,
    Condition,
    BIconCalendar,
    Book,
    ContextMenu,
    IssueDetailsPopover
  },
  mixins: [l10nMixin, collectionMixin, conditionMixin],
  props: {
    publicationcode: {
      type: String,
      required: true
    }
  },
  data: () => ({
    loading: true,
    publicationNameLoading: true,
    filter: {
      missing: true,
      possessed: true
    },
    coverUrl: null,
    issues: null,
    userIssuesForPublication: null,
    userIssuesNotFoundForPublication: [],
    selected: [],
    preselected: [],
    preselectedIndexStart: null,
    preselectedIndexEnd: null,
    hoveredIssueNumber: null,
    currentIssueOpened: null
  }),
  computed: {
    ...mapState("coa", ["publicationNames"]),
    ...mapState("collection", { userIssues: "collection" }),

    country() {
      return this.publicationcode.split("/")[0];
    },
    publicationName() {
      return this.publicationNames[this.publicationcode];
    },

    isTouchScreen: () => window.matchMedia("(pointer: coarse)").matches,
    filteredIssues() {
      const vm = this;
      return this.issues && this.issues.filter(issue =>
        vm.filter.possessed && issue.condition ||
        vm.filter.missing && !issue.condition
      );
    }
  },
  watch: {
    preselectedIndexEnd() {
      this.preselected = this.getPreselected();
    },
    userIssues: {
      immediate: true,
      async handler(newValue) {
        if (newValue) {
          const vm = this;

          this.userIssuesForPublication = newValue.filter(issue =>
            `${issue.country}/${issue.magazine}` === vm.publicationcode)
            .map(issue => ({
                ...issue,
                condition: (vm.conditions.find(({ dbValue }) => dbValue === issue.condition) || { value: "possessed" }).value
              })
            );

          const issuesWithTitles = (await axios.get(`/api/coa/list/issues/withTitle/asArray/${this.publicationcode}`)).data;

          this.issues = issuesWithTitles
            .map(issue => ({
              ...issue,
              ...(vm.userIssuesForPublication.find(({ issueNumber: userIssueNumber }) => userIssueNumber === issue.issueNumber) || {})
            }));
          const coaIssueNumbers = issuesWithTitles.map(({issueNumber}) => issueNumber)
          this.userIssuesNotFoundForPublication = this.userIssuesForPublication
            .filter(({ issueNumber }) => !coaIssueNumbers.includes(issueNumber))
          this.loading = false;
        }
      }
    }
  },
  async mounted() {
    await this.loadPurchases();
    await this.fetchPublicationNames([this.publicationcode]);
    this.publicationNameLoading = false;
  },
  methods: {
    ...mapActions("coa", ["fetchPublicationNames"]),
    ...mapActions("collection", ["loadCollection", "loadPurchases"]),
    openContextMenuIfBookNotOpen(e) {
      if (this.currentIssueOpened === null) {
        this.$refs.contextMenu.$refs.menu.open(e);
      }
    },
    getPreselected() {
      const vm = this;
      if ([this.preselectedIndexStart, this.preselectedIndexEnd].includes(null)) {
        return this.preselected;
      }
      return this.filteredIssues
        .map(({ issueNumber }) => issueNumber)
        .filter((issueNumber, index) =>
          index >= vm.preselectedIndexStart && index <= vm.preselectedIndexEnd
        );
    },
    updateSelected() {
      const vm = this;
      this.selected = this.issues
        .map(({ issueNumber }) => issueNumber)
        .filter(issueNumber => vm.selected.includes(issueNumber) !== vm.preselected.includes(issueNumber));
      this.preselectedIndexStart = this.preselectedIndexEnd = null;
      this.preselected = [];
    },
    async deletePublicationIssues(issuesToDelete) {
      await this.updateIssues({
        publicationCode: this.publicationcode,
        issueNumbers: issuesToDelete.map(({ issueNumber }) => issueNumber),
        condition: this.conditions.find(({ value }) => value === "missing").dbValue,
        istosell: false,
        purchaseId: null
      });
    },
    async updateIssues(data) {
      await axios.post("/api/collection/issues", data);
      await this.loadCollection(true);
    },
    async createPurchase({ date, description }) {
      await axios.post("/api/collection/purchases", {
        date,
        description
      });
      await this.loadPurchases(true);
    }
  }
};
</script>

<style scoped lang="scss">

.issue-list-header {
  border: 0;
  width: 100%;
}

.issue-filter {
  float: right;
  white-space: nowrap;

  * {
    font-size: 11px;
  }

  tr td:nth-child(2) {
    text-align: right;
  }
}

.issue-list {
  clear: both;
  user-select: none;

  .issue {
    width: 100%;
    background-color: black;
    cursor: default;
    height: 20px;

    &:hover {
      opacity: 0.7;
      filter: alpha(opacity=70);
      -moz-opacity: 0.7;
    }

    &.preselected {
      opacity: 0.7;
      filter: alpha(opacity=70);
    }

    &.selected {
      background-repeat: no-repeat;
      background-position: right center;
      outline: 1px solid white;
    }

    &.issue-possessed {
      background-color: rgb(200, 137, 100);
    }

    .preview {
      cursor: pointer;
    }

    .issue-title {
      color: #aaa;
    }

    .issue-details-wrapper {
      display: inline;
      float: right;
      padding-right: 20px;
      padding-top: 2px;

      .issue-details {
        float: right;
        width: 14px;
        height: 14px;
        margin-right: 5px;

        &.issue-condition {
          border-radius: 50%;
        }
      }
    }
  }
}

@media (max-width: 767px) {
  .issue-text {
    white-space: nowrap;
    max-width: 170px;
    overflow-x: hidden;
  }

  .issue-details-wrapper {
    padding-right: 0;
  }
}

</style>
