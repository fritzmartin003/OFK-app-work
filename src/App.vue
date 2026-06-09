<script setup>
import { computed, ref } from 'vue'
import { employees, licenseGuidelines } from './data/employees'

const views = [
  { id: 'employees', label: 'Mitarbeiter' },
  { id: 'costs', label: 'Kosten' },
  { id: 'guide', label: 'Lizenz-Anleitung' },
  { id: 'stats', label: 'Statistik' },
]

const activeView = ref('employees')
const search = ref('')
const departmentFilter = ref('Alle')
const ofkFilter = ref('Alle')
const accountTypeFilter = ref('Alle')
const licenseFilter = ref('Alle')

const formatter = new Intl.NumberFormat('de-DE', {
  style: 'currency',
  currency: 'EUR',
})

const numberFormatter = new Intl.NumberFormat('de-DE')

const enrichedEmployees = computed(() =>
  employees.map((employee) => ({
    ...employee,
    totalCost: employee.licenses.reduce((sum, license) => sum + license.cost, 0),
    licenseNames: employee.licenses.map((license) => license.name),
  })),
)

const departments = computed(() => uniqueOptions(enrichedEmployees.value, 'department'))
const ofks = computed(() => uniqueOptions(enrichedEmployees.value, 'ofk'))
const accountTypes = computed(() => uniqueOptions(enrichedEmployees.value, 'accountType'))
const licenses = computed(() =>
  [...new Set(enrichedEmployees.value.flatMap((employee) => employee.licenseNames))].sort(),
)

const filteredEmployees = computed(() => {
  const term = search.value.trim().toLowerCase()

  return enrichedEmployees.value.filter((employee) => {
    const matchesSearch =
      !term ||
      [
        employee.displayName,
        employee.email,
        employee.department,
        employee.ofk,
        employee.city,
        employee.country,
        employee.accountType,
      ]
        .join(' ')
        .toLowerCase()
        .includes(term)

    const matchesDepartment =
      departmentFilter.value === 'Alle' || employee.department === departmentFilter.value
    const matchesOfk = ofkFilter.value === 'Alle' || employee.ofk === ofkFilter.value
    const matchesAccountType =
      accountTypeFilter.value === 'Alle' || employee.accountType === accountTypeFilter.value
    const matchesLicense =
      licenseFilter.value === 'Alle' || employee.licenseNames.includes(licenseFilter.value)

    return (
      matchesSearch &&
      matchesDepartment &&
      matchesOfk &&
      matchesAccountType &&
      matchesLicense
    )
  })
})

const departmentCosts = computed(() =>
  groupBy(enrichedEmployees.value, 'department').map((group) => ({
    label: group.label,
    employees: group.items.length,
    licenses: group.items.reduce((sum, employee) => sum + employee.licenses.length, 0),
    total: sumCosts(group.items),
  })),
)

const ofkCosts = computed(() =>
  groupBy(enrichedEmployees.value, 'ofk').map((group) => ({
    label: group.label,
    code: group.items[0]?.ofkCode ?? '',
    employees: group.items.length,
    total: sumCosts(group.items),
  })),
)

const licenseStats = computed(() => {
  const map = new Map()

  enrichedEmployees.value.forEach((employee) => {
    employee.licenses.forEach((license) => {
      const current = map.get(license.name) ?? {
        label: license.name,
        count: 0,
        total: 0,
      }
      current.count += 1
      current.total += license.cost
      map.set(license.name, current)
    })
  })

  return [...map.values()].sort((a, b) => b.total - a.total)
})

const accountTypeStats = computed(() =>
  groupBy(enrichedEmployees.value, 'accountType').map((group) => ({
    label: group.label,
    count: group.items.length,
    total: sumCosts(group.items),
  })),
)

const countryStats = computed(() =>
  groupBy(enrichedEmployees.value, 'country').map((group) => ({
    label: group.label,
    count: group.items.length,
    total: sumCosts(group.items),
  })),
)

const totalMonthlyCost = computed(() => sumCosts(enrichedEmployees.value))
const totalLicenses = computed(() =>
  enrichedEmployees.value.reduce((sum, employee) => sum + employee.licenses.length, 0),
)
const averageCost = computed(() => totalMonthlyCost.value / enrichedEmployees.value.length)
const topDepartment = computed(() => departmentCosts.value[0]?.label ?? '-')
const maxDepartmentCost = computed(() =>
  Math.max(...departmentCosts.value.map((department) => department.total), 1),
)
const maxLicenseCost = computed(() =>
  Math.max(...licenseStats.value.map((license) => license.total), 1),
)

const employeeRecommendations = computed(() =>
  enrichedEmployees.value.map((employee) => {
    const matches = licenseGuidelines.filter((guideline) => guideline.condition(employee))
    const required = [...new Set(matches.flatMap((match) => match.required))]
    const optional = [...new Set(matches.flatMap((match) => match.optional))]
    const rationale = matches.map((match) => match.rationale)
    const assigned = employee.licenseNames
    const missing = required.filter(
      (needed) =>
        !assigned.some((license) => needed.toLowerCase().includes(license.toLowerCase())),
    )

    return {
      employee,
      required,
      optional,
      rationale,
      missing,
      status: missing.length ? 'Prüfen' : 'Passend',
    }
  }),
)

function uniqueOptions(list, key) {
  return [...new Set(list.map((item) => item[key]))].sort()
}

function groupBy(list, key) {
  const groups = new Map()

  list.forEach((item) => {
    const label = item[key] || 'Nicht zugeordnet'
    const current = groups.get(label) ?? []
    current.push(item)
    groups.set(label, current)
  })

  return [...groups.entries()]
    .map(([label, items]) => ({ label, items }))
    .sort((a, b) => sumCosts(b.items) - sumCosts(a.items))
}

function sumCosts(list) {
  return list.reduce((sum, employee) => sum + employee.totalCost, 0)
}

function money(value) {
  return formatter.format(value)
}

function count(value) {
  return numberFormatter.format(value)
}
</script>

<template>
  <div class="app-shell">
    <aside class="sidebar">
      <div>
        <p class="app-kicker">OFK Lizenz-Cockpit</p>
        <h1>SAM User Management</h1>
      </div>

      <nav class="nav-tabs" aria-label="Hauptnavigation">
        <button
          v-for="view in views"
          :key="view.id"
          type="button"
          :class="{ active: activeView === view.id }"
          @click="activeView = view.id"
        >
          {{ view.label }}
        </button>
      </nav>
    </aside>

    <main class="workspace">
      <section class="summary-grid" aria-label="Kennzahlen">
        <article class="metric">
          <span>Mitarbeiter</span>
          <strong>{{ count(enrichedEmployees.length) }}</strong>
        </article>
        <article class="metric">
          <span>Monatskosten</span>
          <strong>{{ money(totalMonthlyCost) }}</strong>
        </article>
        <article class="metric">
          <span>Lizenzen</span>
          <strong>{{ count(totalLicenses) }}</strong>
        </article>
        <article class="metric">
          <span>Ø Kosten pro Nutzer</span>
          <strong>{{ money(averageCost) }}</strong>
        </article>
      </section>

      <section v-if="activeView === 'employees'" class="panel">
        <div class="panel-heading">
          <div>
            <p class="section-kicker">Mitarbeiterübersicht</p>
            <h2>Personen, Abteilungen und eingesetzte Lizenzen</h2>
          </div>
          <strong>{{ count(filteredEmployees.length) }} Treffer</strong>
        </div>

        <div class="filters">
          <label>
            Suche
            <input v-model="search" type="search" placeholder="Name, Mail, Ort, Lizenz" />
          </label>
          <label>
            Abteilung
            <select v-model="departmentFilter">
              <option>Alle</option>
              <option v-for="department in departments" :key="department">
                {{ department }}
              </option>
            </select>
          </label>
          <label>
            OFK
            <select v-model="ofkFilter">
              <option>Alle</option>
              <option v-for="ofk in ofks" :key="ofk">{{ ofk }}</option>
            </select>
          </label>
          <label>
            Kontotyp
            <select v-model="accountTypeFilter">
              <option>Alle</option>
              <option v-for="type in accountTypes" :key="type">{{ type }}</option>
            </select>
          </label>
          <label>
            Lizenz
            <select v-model="licenseFilter">
              <option>Alle</option>
              <option v-for="license in licenses" :key="license">{{ license }}</option>
            </select>
          </label>
        </div>

        <div class="table-wrap">
          <table>
            <thead>
              <tr>
                <th>Mitarbeiter</th>
                <th>Abteilung</th>
                <th>OFK</th>
                <th>Standort</th>
                <th>Lizenzen</th>
                <th class="number">Kosten / Monat</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="employee in filteredEmployees" :key="employee.id">
                <td>
                  <strong>{{ employee.displayName }}</strong>
                  <span>{{ employee.email }}</span>
                </td>
                <td>{{ employee.department }}</td>
                <td>
                  <strong>{{ employee.ofkCode }}</strong>
                  <span>{{ employee.ofk }}</span>
                </td>
                <td>{{ employee.city }}, {{ employee.country }}</td>
                <td>
                  <div class="chips">
                    <span v-for="license in employee.licenses" :key="license.name">
                      {{ license.name }}
                    </span>
                  </div>
                </td>
                <td class="number">{{ money(employee.totalCost) }}</td>
              </tr>
            </tbody>
          </table>
        </div>
      </section>

      <section v-if="activeView === 'costs'" class="panel">
        <div class="panel-heading">
          <div>
            <p class="section-kicker">Kosten-Tracking</p>
            <h2>Abteilungskosten und Lizenzverteilung</h2>
          </div>
          <strong>Top: {{ topDepartment }}</strong>
        </div>

        <div class="split-grid">
          <article class="chart-panel">
            <h3>Abteilungen</h3>
            <div class="bar-list">
              <div v-for="department in departmentCosts" :key="department.label" class="bar-row">
                <div>
                  <strong>{{ department.label }}</strong>
                  <span>{{ department.employees }} MA · {{ department.licenses }} Lizenzen</span>
                </div>
                <div class="bar-track">
                  <span :style="{ width: `${(department.total / maxDepartmentCost) * 100}%` }" />
                </div>
                <strong>{{ money(department.total) }}</strong>
              </div>
            </div>
          </article>

          <article class="chart-panel">
            <h3>Lizenzen</h3>
            <div class="bar-list">
              <div v-for="license in licenseStats" :key="license.label" class="bar-row">
                <div>
                  <strong>{{ license.label }}</strong>
                  <span>{{ license.count }} Zuweisungen</span>
                </div>
                <div class="bar-track accent">
                  <span :style="{ width: `${(license.total / maxLicenseCost) * 100}%` }" />
                </div>
                <strong>{{ money(license.total) }}</strong>
              </div>
            </div>
          </article>
        </div>

        <div class="table-wrap compact">
          <table>
            <thead>
              <tr>
                <th>OFK</th>
                <th>Kürzel</th>
                <th class="number">Mitarbeiter</th>
                <th class="number">Monatskosten</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="ofk in ofkCosts" :key="ofk.label">
                <td>{{ ofk.label }}</td>
                <td>{{ ofk.code }}</td>
                <td class="number">{{ count(ofk.employees) }}</td>
                <td class="number">{{ money(ofk.total) }}</td>
              </tr>
            </tbody>
          </table>
        </div>
      </section>

      <section v-if="activeView === 'guide'" class="panel">
        <div class="panel-heading">
          <div>
            <p class="section-kicker">Lizenz-Anleitung</p>
            <h2>Empfehlung je Mitarbeiter</h2>
          </div>
          <strong>{{ count(employeeRecommendations.length) }} Profile</strong>
        </div>

        <div class="recommendation-list">
          <article
            v-for="recommendation in employeeRecommendations"
            :key="recommendation.employee.id"
            class="recommendation"
          >
            <div class="recommendation-head">
              <div>
                <h3>{{ recommendation.employee.displayName }}</h3>
                <span>
                  {{ recommendation.employee.accountType }} ·
                  {{ recommendation.employee.department }}
                </span>
              </div>
              <span :class="['status-pill', recommendation.status === 'Passend' ? 'ok' : 'warn']">
                {{ recommendation.status }}
              </span>
            </div>

            <div class="recommendation-grid">
              <div>
                <h4>Aktuell eingesetzt</h4>
                <p>{{ recommendation.employee.licenseNames.join(', ') }}</p>
              </div>
              <div>
                <h4>Benötigt</h4>
                <p>{{ recommendation.required.join(', ') }}</p>
              </div>
              <div>
                <h4>Optional prüfen</h4>
                <p>{{ recommendation.optional.join(', ') || 'Keine optionale Lizenz' }}</p>
              </div>
            </div>

            <p class="reason">{{ recommendation.rationale.join(' ') }}</p>
          </article>
        </div>
      </section>

      <section v-if="activeView === 'stats'" class="panel">
        <div class="panel-heading">
          <div>
            <p class="section-kicker">Statistik</p>
            <h2>Sinnvolle Auswertungen aus der Excel-Basis</h2>
          </div>
          <strong>{{ money(totalMonthlyCost * 12) }} / Jahr</strong>
        </div>

        <div class="stats-grid">
          <article class="chart-panel">
            <h3>Kontotypen</h3>
            <dl class="stat-list">
              <div v-for="type in accountTypeStats" :key="type.label">
                <dt>{{ type.label }}</dt>
                <dd>{{ type.count }} Konten · {{ money(type.total) }}</dd>
              </div>
            </dl>
          </article>

          <article class="chart-panel">
            <h3>Länder</h3>
            <dl class="stat-list">
              <div v-for="country in countryStats" :key="country.label">
                <dt>{{ country.label }}</dt>
                <dd>{{ country.count }} Mitarbeiter · {{ money(country.total) }}</dd>
              </div>
            </dl>
          </article>

          <article class="chart-panel wide">
            <h3>Kostenkennzahlen</h3>
            <div class="insight-grid">
              <div>
                <span>Teuerster Mitarbeiter</span>
                <strong>
                  {{
                    enrichedEmployees.reduce((max, employee) =>
                      employee.totalCost > max.totalCost ? employee : max,
                    ).displayName
                  }}
                </strong>
              </div>
              <div>
                <span>Häufigste Lizenz</span>
                <strong>{{ licenseStats[0]?.label }}</strong>
              </div>
              <div>
                <span>Abteilungen</span>
                <strong>{{ count(departmentCosts.length) }}</strong>
              </div>
              <div>
                <span>Standorte</span>
                <strong>{{ count(new Set(enrichedEmployees.map((item) => item.city)).size) }}</strong>
              </div>
            </div>
          </article>
        </div>
      </section>
    </main>
  </div>
</template>
