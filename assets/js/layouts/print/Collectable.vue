<template>
  <div v-if="ready">
    <table class="collectable">
      <tr
        v-for="line in lines"
        :key="line"
      >
        <td />
        <td
          v-for="subrange in numbersPerRow"
          :key="subrange"
        >
          {{ (line - 1) * numbersPerRow + subrange }}
        </td>
        <td
          v-if="line === 1"
          :rowspan="lines"
          class="total_ligne"
        >
          {{ l10n.TOTAL }}
        </td>
      </tr>
      <template v-for="(_, publicationCode) in publicationNames">
        <tr
          v-for="line in lines"
          :key="`${publicationCode}-line${line}`"
        >
          <td
            v-if="line === 1"
            :rowspan="lines+1"
            class="libelle_ligne"
          >
            <img
              :alt="publicationCode.split('/')[0]"
              :src="`${imagePath}/flags/${publicationCode.split('/')[0]}.png`"
            >
            <br>
            {{ publicationCode.split('/')[1] }}
            <br>
          </td>
          <td
            v-for="subrange in numbersPerRow"
            :key="subrange"
          >
            <span
              v-for="letter in issuesPerCell[publicationCode][(line-1)*numbersPerRow+subrange]"
              :key="letter"
              class="letter"
            >{{
              letter
            }}</span>
          </td>
          <td
            v-if="line === 1"
            class="total_ligne"
            :rowspan="lines+1"
          >
            {{ totalPerPublication[publicationCode] }}
          </td>
        </tr>
        <tr :key="`${publicationCode}-others`">
          <td
            v-if="issuesPerCell[publicationCode]['non-numeric'].length"
            :colspan="numbersPerRow"
          >
            Autres :
            {{ issuesPerCell[publicationCode]['non-numeric'].join(', ') }}
          </td>
        </tr>
      </template>
    </table>
    <table class="legendes">
      <tr>
        <td class="legende_numeros">
          <table>
            <tr>
              <td
                align="center"
                colspan="2"
              >
                <u>{{
                  l10n.NUMEROS[0].toUpperCase() + l10n.NUMEROS.substring(1, l10n.NUMEROS.length).toLowerCase()
                }}</u>
              </td>
            </tr>
            <tr
              v-for="(_, i) in Math.floor(letterToNumber(maxLetter) / 6)"
              :key="i"
            >
              <td
                v-for="(__, group) in letterToNumber(maxLetter)"
                :key="group"
              >
                <span v-if="Math.floor(group / 6) === i">
                  {{ numberToLetter(group) }} : {{ group * 100 + 1 }}-&gt;{{ (group + 1) * 100 }}
                </span>
              </td>
            </tr>
          </table>
        </td>
        <td class="legende_magazines">
          <table>
            <tr>
              <td
                align="center"
                colspan="4"
              >
                <u>{{ l10n.PUBLICATIONS }}</u>
              </td>
            </tr>
            <tr
              v-for="(publicationName, publicationCode) in publicationNames"
              :key="publicationCode"
            >
              <td>
                <img
                  :alt="publicationCode.split('/')[0]"
                  :src="`${imagePath}/flags/${publicationCode.split('/')[0]}.png`"
                >
                {{ publicationCode.split('/')[1] }}
              </td>
              <td>{{ publicationName }}</td>
            </tr>
          </table>
        </td>
      </tr>
    </table>
  </div>
  <div v-else-if="l10n">
    {{ l10n.CHARGEMENT }}
  </div>
</template>

<script>
import collectionMixin from "../../mixins/collectionMixin";
import l10nMixin from "../../mixins/l10nMixin";
import {mapActions} from "vuex";

const doubleNumberRegex = /^(\d{1,2})(\d{2})-(\d{2})$/

export default {
  mixins: [collectionMixin, l10nMixin],
  data() {
    const lines = 2
    return {
      lines,
      numbersPerRow: 100 / lines,
      issuesPerCell: null
    }
  },
  computed: {
    imagePath: () => window.imagePath,
    ready() {
      return this.issuesPerCell && this.countryNames && this.publicationNames && this.l10n
    },
    maxLetter() {
      return !this.issuesPerCell ? null : this.numberToLetter([
            ...new Set(JSON.stringify(this.issuesPerCell)
                .match(/(?<=")[a-zA-Z]+(?=")/g))
          ]
              .map(this.letterToNumber)
              .sort((a, b) => b - a)[0]
      )
    },
  },
  watch: {
    collection(newCollectionValue) {
      const vm = this
      const addIssueToCell = (acc, publicationCode, issueNumber, isDoubleIssueStart, isDoubleIssueEnd) => {
        let mod, number
        if (Number.isNaN(issueNumber % 100)) {
          mod = 'non-numeric'
          number = issueNumber
        } else {
          mod = issueNumber % 100
          number = (issueNumber - mod) / 100
        }
        if (!acc[publicationCode]) {
          acc[publicationCode] = {
            'non-numeric': []
          }
        }
        if (!acc[publicationCode][mod]) {
          acc[publicationCode][mod] = []
        }

        acc[publicationCode][mod].push([
          isDoubleIssueEnd ? '<' : '',
          typeof number === 'string' ? number : vm.numberToLetter(number),
          isDoubleIssueStart ? '>' : ''
        ].join(''))
      }
      this.issuesPerCell = newCollectionValue.reduce((acc, issue) => {
        const publicationCode = `${issue.country}/${issue.magazine}`
        const issueNumber = issue.issueNumber;
        const doubleNumberMatch = issueNumber.match(doubleNumberRegex)
        if (doubleNumberMatch && parseInt(doubleNumberMatch[2]) === parseInt(doubleNumberMatch[1]) + 1) {
          const [, part1, issue1, issue2] = doubleNumberMatch
          addIssueToCell(acc, publicationCode, `${part1}${issue1}`, true)
          addIssueToCell(acc, publicationCode, `${part1}${issue2}`, false, true)
        } else {
          addIssueToCell(acc, publicationCode, issueNumber)
        }
        return acc
      }, {});
    },
    totalPerPublication(newValue) {
      this.fetchPublicationNames(Object.keys(newValue))
    }
  },
  async mounted() {
    await this.fetchCountryNames()
    await this.loadPurchases()
  },

  methods: {
    ...mapActions("coa", ["fetchCountryNames", "fetchPublicationNames"]),
    ...mapActions("collection", ["loadPurchases"]),
    numberToLetter: number => String.fromCharCode((number < 26 ? "a".charCodeAt() : "A".charCodeAt() - 26) + number),
    letterToNumber: letter => letter >= "a" ? letter.charCodeAt() - "a".charCodeAt() : 26 + letter.charCodeAt() - "A".charCodeAt()
  }
}
</script>

<style scoped lang="scss">
table {
  color: black;
  font: 11px/15px verdana, arial, sans-serif;

  &.collectable, &.legendes {
    width: 90%;
  }
}

td {
  &.legende_numeros, &.legende_magazines, &.achats {
    vertical-align: top;
    border-left: 1px solid gray;
    padding: 8px;
  }
}

table.collectable {
  border: solid 1px black;
  border-collapse: collapse;

  tr {
    height: 15px;

    &:empty {
      height: 0;
    }

    td {
      text-align: left;
      border: solid 1px black;
      vertical-align: top;
      min-width: 25px;
      max-width: 25px;
      word-wrap: break-word;

      &.libelle_ligne, &.total_ligne {
        text-align: center;
        vertical-align: middle;
        max-width: none;
        white-space: nowrap;
      }
    }
  }
}
</style>