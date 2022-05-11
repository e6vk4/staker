<template>
  <div class="text-slate-800">
    <!-- Header and Filters -->
    <div class="sticky top-0 left-0 z-20 flex justify-center w-full">
      <div class="w-full max-w-3xl px-20 pt-8 pb-6 bg-slate-200">
        <!-- Keyword Search -->
        <input
          v-model="searchQuery"
          type="text"
          class="w-full h-10 px-4 mb-6 rounded focus:outline-0"
          placeholder="Search by Ticker Symbol or Company name"
        />

        <!-- Issues Filters -->
        <div class="grid grid-cols-2">
          <div
            v-for="(issue, indexA) in issues"
            :key="indexA"
            class="flex items-center justify-between px-3 py-1 mb-3 mr-3 rounded-full"
            :class="`${issuesColorMap[issue]}`"
          >
            <span class="font-mono text-sm">{{ issue }}</span>
            <input
              :id="issue"
              class="cursor-pointer"
              :checked="filteredIssues.includes(issue)"
              type="checkbox"
              @input="updateFilteredIssues"
            />
          </div>
        </div>

        <!-- Results Summary -->
        <div
          v-if="searchQuery || filteredIssues.length"
          class="mt-5 text-sm text-center text-slate-600"
        >
          found {{ filteredCompaniesList.length }} matching results
        </div>
      </div>
    </div>

    <!-- Companies List -->
    <div class="flex justify-center w-screen">
      <div class="w-full max-w-3xl px-20 py-10 bg-slate-100">
        <!-- Content -->
        <div>
          <div
            v-for="(company, indexB) in paginatedCompaniesList"
            :key="indexB"
            class="px-4 py-6 bg-white rounded-md mb-7"
          >
            <div :class="hasIssues(company['Ticker Symbol']) ? 'mb-4' : 'mb-0'">
              <span class="mr-3 font-medium">{{
                company['Company Name']
              }}</span>
              <span
                class="px-2 py-1 mr-3 font-mono text-xs rounded bg-slate-200"
                >{{ company['Ticker Symbol'] }}</span
              >
              <IconCaret class="mr-3 -rotate-90" />
              <span class="font-mono text-xs">{{
                getIssuesCountLabel(company['Ticker Symbol'])
              }}</span>
            </div>
            <div>
              <span
                v-for="(issue, indexC) in getIssues(company['Ticker Symbol'])"
                :key="indexC"
                class="px-3 py-1 mr-3 font-mono text-sm rounded-full"
                :class="`${issuesColorMap[issue]}`"
              >
                {{ issue }}
              </span>
            </div>
          </div>
        </div>
        <!-- Pagination -->
        <div class="flex items-center justify-center">
          <button
            :class="start === 0 ? 'cursor-not-allowed opacity-30' : ''"
            class="px-3 py-1 mr-8 border rounded hover:bg-slate-200"
            @click="paginate('prev')"
          >
            Prev
          </button>
          <button
            :class="
              start >= filteredCompaniesList.length
                ? 'cursor-not-allowed opacity-30'
                : ''
            "
            class="px-3 py-1 border rounded hover:bg-slate-200"
            @click="paginate('next')"
          >
            Next
          </button>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import IconCaret from '../components/Icons/IconCaret.vue'

export default {
  name: 'HomePage',
  components: { IconCaret },
  async asyncData({ $content }) {
    return {
      companyExclusions: await $content('company_exclusions').fetch(),
    }
  },
  data() {
    return {
      searchQuery: '',
      start: 0,
      filteredIssues: [],
      issues: [
        'Animal Testing',
        'Nuclear Weapons',
        'Coal Power',
        'Rainforest Destruction',
      ],
      issuesColorMap: {
        'Animal Testing': 'bg-purple-200',
        'Nuclear Weapons': 'bg-blue-200',
        'Coal Power': 'bg-yellow-200',
        'Rainforest Destruction': 'bg-teal-200',
      },
    }
  },

  computed: {
    // Creates a list of companies from the CSV.
    companiesList() {
      const companiesList = [...this.companyExclusions?.body]
      return companiesList
    },
    // Creates a map of ticker symbols to issues.
    tickerIssuesMap() {
      const companiesList = [...this.companiesList]
      const result = companiesList.map((company) => {
        const issues = this.issues.filter((issue) => company[issue] === 'TRUE')
        const ticker = company['Ticker Symbol']
        return {
          ticker,
          issues,
        }
      })
      return result
    },
    // Filters the companies list based on the search query and the selected issues.
    filteredCompaniesList() {
      const companiesList = [...this.companiesList]
      const filteredCompaniesList = companiesList
        .filter((company) => {
          if (!this.searchQuery) return true
          const name = company['Company Name'].toLowerCase()
          const ticker = company['Ticker Symbol'].toLowerCase()
          const query = this.searchQuery.toLowerCase()
          return name.includes(query) || ticker.includes(query)
        })
        .filter((company) => {
          const filteredIssues = [...this.filteredIssues]
          if (filteredIssues.length) {
            const ticker = company['Ticker Symbol']
            const { issues } = this.tickerIssuesMap.find(
              (item) => item.ticker === ticker
            )
            return !!filteredIssues.some((issue) => issues.includes(issue))
          } else return true
        })
      return filteredCompaniesList
    },
    // Creates a list of companies to be displayed on the page.
    paginatedCompaniesList() {
      const filteredCompaniesList = [...this.filteredCompaniesList]
      const start = this.start
      const end = this.start + 50
      const paginatedCompaniesList = filteredCompaniesList.slice(start, end)
      return paginatedCompaniesList
    },
  },
  watch: {
    searchQuery: {
      handler() {
        this.start = 0
      },
    },
    filteredIssues: {
      handler() {
        this.start = 0
      },
    },
  },
  methods: {
    // Returns the issues for a given ticker symbol.
    getIssues(ticker) {
      const result = this.tickerIssuesMap.find((item) => item.ticker === ticker)
      return result.issues
    },
    // Returns a text label for the number of issues a company has.
    getIssuesCountLabel(ticker) {
      const result = this.getIssues(ticker)
      const length = result.length
      let label = `${length} issues`
      if (length === 1) label = `${length} issue`
      return label
    },
    // Checks if a company has any issues.
    hasIssues(ticker) {
      const result = this.getIssues(ticker)
      return !!result.length
    },
    // Updates the `filteredIssues` array.
    updateFilteredIssues(e) {
      const issue = e.target.id
      const index = this.filteredIssues.findIndex((x) => x === issue)
      if (index === -1) this.filteredIssues.push(issue)
      else this.filteredIssues.splice(index, 1)
    },
    // Updates the `start` variable for pagination.
    paginate(dir) {
      if (dir === 'next') {
        if (this.start >= this.filteredCompaniesList.length) return
        this.start = this.start + 50
      } else {
        if (this.start === 0) return
        this.start = this.start - 50
      }
      this.scrollToTop()
    },
    // Scrolls the page to the top.
    scrollToTop() {
      window.scrollTo({ top: 0, behavior: 'smooth' })
    },
  },
}
</script>

<style lang="scss">
html {
  scroll-behavior: smooth;
}
</style>
