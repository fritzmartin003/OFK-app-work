<script setup>
import { computed, ref } from 'vue'
import employeesCsv from './data/employees.csv?raw'

const views = [
  { id: 'stats', label: 'Statistiken' },
  { id: 'employees', label: 'Mitarbeiter' },
  { id: 'guide', label: 'Lizenzanleitung' },
]

const activeView = ref('stats')

const statsFilters = ref({
  search: '',
  department: 'Alle',
  city: 'Alle',
  country: 'Alle',
  license: 'Alle',
  accountType: 'Alle',
  ofk: 'Alle',
})

const employeeFilters = ref({
  search: '',
  department: 'Alle',
  ofk: 'Alle',
  accountType: 'Alle',
  license: 'Alle',
})

const formatter = new Intl.NumberFormat('de-DE', {
  style: 'currency',
  currency: 'EUR',
})

const numberFormatter = new Intl.NumberFormat('de-DE')
const employees = parseEmployeesCsv(employeesCsv)

const enrichedEmployees = computed(() =>
  employees.map((employee) => ({
    ...employee,
    totalCost: employee.licenses.reduce((sum, license) => sum + license.cost, 0),
    licenseNames: employee.licenses.map((license) => license.name),
    location: `${employee.city}, ${employee.country}`,
  })),
)

const departments = computed(() => uniqueOptions(enrichedEmployees.value, 'department'))
const cities = computed(() => uniqueOptions(enrichedEmployees.value, 'city'))
const countries = computed(() => uniqueOptions(enrichedEmployees.value, 'country'))
const ofks = computed(() => uniqueOptions(enrichedEmployees.value, 'ofk'))
const accountTypes = computed(() => uniqueOptions(enrichedEmployees.value, 'accountType'))
const licenses = computed(() =>
  [...new Set(enrichedEmployees.value.flatMap((employee) => employee.licenseNames))].sort(),
)

const filteredStatsEmployees = computed(() => filterEmployees(enrichedEmployees.value, statsFilters.value))
const filteredEmployees = computed(() => filterEmployees(enrichedEmployees.value, employeeFilters.value))

const totalMonthlyCost = computed(() => sumCosts(filteredStatsEmployees.value))
const totalLicenses = computed(() => countLicenses(filteredStatsEmployees.value))
const averageCost = computed(() =>
  filteredStatsEmployees.value.length ? totalMonthlyCost.value / filteredStatsEmployees.value.length : 0,
)
const annualCost = computed(() => totalMonthlyCost.value * 12)

const licenseOverview = computed(() => {
  const map = new Map()

  filteredStatsEmployees.value.forEach((employee) => {
    employee.licenses.forEach((license) => {
      const current = map.get(license.name) ?? {
        label: license.name,
        assignments: 0,
        total: 0,
        departments: new Set(),
        locations: new Set(),
      }

      current.assignments += 1
      current.total += license.cost
      current.departments.add(employee.department)
      current.locations.add(employee.location)
      map.set(license.name, current)
    })
  })

  return [...map.values()]
    .map((item) => ({
      ...item,
      departments: item.departments.size,
      locations: item.locations.size,
      average: item.assignments ? item.total / item.assignments : 0,
    }))
    .sort((a, b) => b.total - a.total)
})

const departmentOverview = computed(() =>
  groupEmployees(filteredStatsEmployees.value, (employee) => employee.department).map((group) => ({
    label: group.label,
    employees: group.items.length,
    licenses: countLicenses(group.items),
    total: sumCosts(group.items),
  })),
)

const locationOverview = computed(() =>
  groupEmployees(filteredStatsEmployees.value, (employee) => employee.location).map((group) => ({
    label: group.label,
    employees: group.items.length,
    licenses: countLicenses(group.items),
    total: sumCosts(group.items),
  })),
)

const departmentLocationOverview = computed(() =>
  groupEmployees(
    filteredStatsEmployees.value,
    (employee) => `${employee.department}|||${employee.location}`,
  ).map((group) => {
    const [department, location] = group.label.split('|||')

    return {
      department,
      location,
      employees: group.items.length,
      licenses: countLicenses(group.items),
      total: sumCosts(group.items),
      topLicense: topLicense(group.items),
    }
  }),
)

const costBreakdown = computed(() => [
  { label: 'Monatliche Gesamtkosten', value: money(totalMonthlyCost.value) },
  { label: 'Jährliche Hochrechnung', value: money(annualCost.value) },
  { label: 'Gefilterte Mitarbeiter', value: count(filteredStatsEmployees.value.length) },
  { label: 'Lizenzzuweisungen', value: count(totalLicenses.value) },
  { label: 'Ø Kosten pro Mitarbeiter', value: money(averageCost.value) },
])

const maxDepartmentCost = computed(() =>
  Math.max(...departmentOverview.value.map((department) => department.total), 1),
)
const maxLocationCost = computed(() =>
  Math.max(...locationOverview.value.map((location) => location.total), 1),
)

function filterEmployees(list, filters) {
  const term = filters.search.trim().toLowerCase()

  return list.filter((employee) => {
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
        ...employee.licenseNames,
      ]
        .join(' ')
        .toLowerCase()
        .includes(term)

    const matchesDepartment = filters.department === 'Alle' || employee.department === filters.department
    const matchesCity = !filters.city || filters.city === 'Alle' || employee.city === filters.city
    const matchesCountry =
      !filters.country || filters.country === 'Alle' || employee.country === filters.country
    const matchesOfk = filters.ofk === 'Alle' || employee.ofk === filters.ofk
    const matchesAccountType =
      filters.accountType === 'Alle' || employee.accountType === filters.accountType
    const matchesLicense = filters.license === 'Alle' || employee.licenseNames.includes(filters.license)

    return (
      matchesSearch &&
      matchesDepartment &&
      matchesCity &&
      matchesCountry &&
      matchesOfk &&
      matchesAccountType &&
      matchesLicense
    )
  })
}

function parseEmployeesCsv(csv) {
  const rows = csv
    .trim()
    .split(/\r?\n/)
    .map((line) => line.split(';').map((cell) => cell.trim()))
  const headers = rows[0]

  return rows.slice(1).map((row) => {
    const record = Object.fromEntries(headers.map((header, index) => [header, row[index] ?? '']))
    const licenses = []

    for (let index = 0; index < 5; index += 1) {
      const suffix = index === 0 ? '' : ` ${index + 1}`
      const name = record[`Licenses${suffix}`]
      const cost = Number.parseFloat(record[`Cost${suffix}`] || '0')

      if (name) {
        licenses.push({
          name,
          cost: Number.isFinite(cost) ? cost : 0,
        })
      }
    }

    return {
      id: slugify(record['User principal name'] || record['Display name']),
      displayName: record['Display name'],
      email: record['User principal name'],
      accountType: record['Account Type'],
      firstName: record['First name'],
      lastName: record['Last name'],
      department: record.Department,
      ofkCode: record['OFK - Kuerzel'],
      ofk: record.OFK,
      city: record.City,
      country: record.CountryOrRegion,
      licenses,
    }
  })
}

function slugify(value) {
  return value
    .toLowerCase()
    .replace(/[^a-z0-9]+/g, '-')
    .replace(/^-|-$/g, '')
}

function uniqueOptions(list, key) {
  return [...new Set(list.map((item) => item[key]))].sort()
}

function groupEmployees(list, getLabel) {
  const groups = new Map()

  list.forEach((item) => {
    const label = getLabel(item) || 'Nicht zugeordnet'
    const current = groups.get(label) ?? []
    current.push(item)
    groups.set(label, current)
  })

  return [...groups.entries()]
    .map(([label, items]) => ({ label, items }))
    .sort((a, b) => sumCosts(b.items) - sumCosts(a.items))
}

function countLicenses(list) {
  return list.reduce((sum, employee) => sum + employee.licenses.length, 0)
}

function sumCosts(list) {
  return list.reduce((sum, employee) => sum + employee.totalCost, 0)
}

function topLicense(list) {
  const map = new Map()

  list.forEach((employee) => {
    employee.licenses.forEach((license) => {
      map.set(license.name, (map.get(license.name) ?? 0) + 1)
    })
  })

  return [...map.entries()].sort((a, b) => b[1] - a[1])[0]?.[0] ?? '-'
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
      <section v-if="activeView === 'stats'" class="page-stack">
        <div class="summary-grid">
          <article class="metric">
            <span>Gesamtkosten / Monat</span>
            <strong>{{ money(totalMonthlyCost) }}</strong>
          </article>
          <article class="metric">
            <span>Gesamtkosten / Jahr</span>
            <strong>{{ money(annualCost) }}</strong>
          </article>
          <article class="metric">
            <span>Mitarbeiter im Filter</span>
            <strong>{{ count(filteredStatsEmployees.length) }}</strong>
          </article>
          <article class="metric">
            <span>Lizenzzuweisungen</span>
            <strong>{{ count(totalLicenses) }}</strong>
          </article>
        </div>

        <section class="panel">
          <div class="panel-heading">
            <div>
              <p class="section-kicker">Statistiken</p>
              <h2>Filterbare Kosten- und Lizenzübersichten</h2>
            </div>
            <strong>{{ count(filteredStatsEmployees.length) }} Datensätze</strong>
          </div>

          <div class="filters stats-filter-grid">
            <label>
              Suche
              <input v-model="statsFilters.search" type="search" placeholder="Name, Mail, Lizenz, Ort" />
            </label>
            <label>
              Abteilung
              <select v-model="statsFilters.department">
                <option>Alle</option>
                <option v-for="department in departments" :key="department">{{ department }}</option>
              </select>
            </label>
            <label>
              Standort
              <select v-model="statsFilters.city">
                <option>Alle</option>
                <option v-for="city in cities" :key="city">{{ city }}</option>
              </select>
            </label>
            <label>
              Land
              <select v-model="statsFilters.country">
                <option>Alle</option>
                <option v-for="country in countries" :key="country">{{ country }}</option>
              </select>
            </label>
            <label>
              Lizenz
              <select v-model="statsFilters.license">
                <option>Alle</option>
                <option v-for="license in licenses" :key="license">{{ license }}</option>
              </select>
            </label>
            <label>
              OFK
              <select v-model="statsFilters.ofk">
                <option>Alle</option>
                <option v-for="ofk in ofks" :key="ofk">{{ ofk }}</option>
              </select>
            </label>
            <label>
              Kontotyp
              <select v-model="statsFilters.accountType">
                <option>Alle</option>
                <option v-for="type in accountTypes" :key="type">{{ type }}</option>
              </select>
            </label>
          </div>
        </section>

        <section class="panel">
          <div class="panel-heading">
            <div>
              <p class="section-kicker">Lizenzübersicht</p>
              <h2>Alle Lizenzen im gefilterten Bestand</h2>
            </div>
            <strong>{{ count(licenseOverview.length) }} Lizenztypen</strong>
          </div>

          <div class="table-wrap">
            <table>
              <thead>
                <tr>
                  <th>Lizenz</th>
                  <th class="number">Zuweisungen</th>
                  <th class="number">Kosten / Monat</th>
                  <th class="number">Ø Kosten</th>
                  <th class="number">Abteilungen</th>
                  <th class="number">Standorte</th>
                </tr>
              </thead>
              <tbody>
                <tr v-for="license in licenseOverview" :key="license.label">
                  <td><strong>{{ license.label }}</strong></td>
                  <td class="number">{{ count(license.assignments) }}</td>
                  <td class="number">{{ money(license.total) }}</td>
                  <td class="number">{{ money(license.average) }}</td>
                  <td class="number">{{ count(license.departments) }}</td>
                  <td class="number">{{ count(license.locations) }}</td>
                </tr>
              </tbody>
            </table>
          </div>
        </section>

        <section class="panel">
          <div class="panel-heading">
            <div>
              <p class="section-kicker">Gesamtkosten</p>
              <h2>Kostenüberblick des gefilterten Bestands</h2>
            </div>
            <strong>{{ money(totalMonthlyCost) }} / Monat</strong>
          </div>

          <div class="cost-tile-grid">
            <article v-for="item in costBreakdown" :key="item.label" class="cost-tile">
              <span>{{ item.label }}</span>
              <strong>{{ item.value }}</strong>
            </article>
          </div>
        </section>

        <section class="split-grid">
          <article class="chart-panel">
            <h3>Kosten und Lizenzen nach Abteilung</h3>
            <div class="bar-list">
              <div v-for="department in departmentOverview" :key="department.label" class="bar-row">
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
            <h3>Kosten und Lizenzen nach Standort</h3>
            <div class="bar-list">
              <div v-for="location in locationOverview" :key="location.label" class="bar-row">
                <div>
                  <strong>{{ location.label }}</strong>
                  <span>{{ location.employees }} MA · {{ location.licenses }} Lizenzen</span>
                </div>
                <div class="bar-track accent">
                  <span :style="{ width: `${(location.total / maxLocationCost) * 100}%` }" />
                </div>
                <strong>{{ money(location.total) }}</strong>
              </div>
            </div>
          </article>
        </section>

        <section class="panel">
          <div class="panel-heading">
            <div>
              <p class="section-kicker">Abteilung x Standort</p>
              <h2>Kosten- und Lizenzverteilung je Abteilung mit Standortunterteilung</h2>
            </div>
            <strong>{{ count(departmentLocationOverview.length) }} Kombinationen</strong>
          </div>

          <div class="table-wrap">
            <table>
              <thead>
                <tr>
                  <th>Abteilung</th>
                  <th>Standort</th>
                  <th class="number">Mitarbeiter</th>
                  <th class="number">Lizenzen</th>
                  <th>Häufigste Lizenz</th>
                  <th class="number">Kosten / Monat</th>
                </tr>
              </thead>
              <tbody>
                <tr v-for="row in departmentLocationOverview" :key="`${row.department}-${row.location}`">
                  <td><strong>{{ row.department }}</strong></td>
                  <td>{{ row.location }}</td>
                  <td class="number">{{ count(row.employees) }}</td>
                  <td class="number">{{ count(row.licenses) }}</td>
                  <td>{{ row.topLicense }}</td>
                  <td class="number">{{ money(row.total) }}</td>
                </tr>
              </tbody>
            </table>
          </div>
        </section>
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
            <input v-model="employeeFilters.search" type="search" placeholder="Name, Mail, Ort, Lizenz" />
          </label>
          <label>
            Abteilung
            <select v-model="employeeFilters.department">
              <option>Alle</option>
              <option v-for="department in departments" :key="department">{{ department }}</option>
            </select>
          </label>
          <label>
            OFK
            <select v-model="employeeFilters.ofk">
              <option>Alle</option>
              <option v-for="ofk in ofks" :key="ofk">{{ ofk }}</option>
            </select>
          </label>
          <label>
            Kontotyp
            <select v-model="employeeFilters.accountType">
              <option>Alle</option>
              <option v-for="type in accountTypes" :key="type">{{ type }}</option>
            </select>
          </label>
          <label>
            Lizenz
            <select v-model="employeeFilters.license">
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
                <td>{{ employee.location }}</td>
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

      <section v-if="activeView === 'guide'" class="panel guide-panel">
        <div class="panel-heading">
          <div>
            <p class="section-kicker">Lizenzanleitung</p>
            <h2>Richtlinie für die passende Lizenzvergabe</h2>
          </div>
        </div>

        <div class="guide-grid">
          <article class="guide-block">
            <h3>1. Bedarf vor Paketgröße</h3>
            <p>
              Die Lizenz wird nach Rolle, Zugriff, Arbeitsweise und Schutzbedarf vergeben. Ein großes
              Paket ist nur dann richtig, wenn die enthaltenen Funktionen regelmäßig genutzt werden oder
              ein klarer Compliance- beziehungsweise Sicherheitsbedarf besteht.
            </p>
          </article>

          <article class="guide-block">
            <h3>2. Interne Standardnutzer</h3>
            <p>
              Für reguläre interne Nutzer ist Microsoft 365 F3 die schlanke Basis. Microsoft 365 E5 wird
              eingesetzt, wenn erweiterte Security-, Compliance-, Analyse- oder Collaboration-Funktionen
              erforderlich sind.
            </p>
          </article>

          <article class="guide-block">
            <h3>3. Externe Nutzer</h3>
            <p>
              Externe Nutzer erhalten nur die Lizenzen, die für Identität, Zugriff und Kommunikation
              nötig sind. Entra ID P2 und Exchange Online können ausreichen, wenn keine vollständige
              Office- oder Collaboration-Umgebung benötigt wird.
            </p>
          </article>

          <article class="guide-block">
            <h3>4. Service Accounts</h3>
            <p>
              Service Accounts werden restriktiv lizenziert. Interaktive Microsoft-365-Pakete sind nur
              zulässig, wenn der Account tatsächlich eine interaktive Nutzung hat. Sicherheits-Add-ons
              werden bevorzugt nach technischem Schutzbedarf vergeben.
            </p>
          </article>

          <article class="guide-block">
            <h3>5. Zusatzlizenzen</h3>
            <p>
              Copilot, Visio, Planner und vergleichbare Add-ons werden nur bei konkretem fachlichen
              Einsatz vergeben. Die Vergabe sollte befristet überprüft werden, wenn Nutzung oder Rolle
              noch unklar sind.
            </p>
          </article>

          <article class="guide-block">
            <h3>6. Regelmäßige Prüfung</h3>
            <p>
              OFKs sollten monatlich prüfen, ob Lizenzkosten, Standortwechsel, Abteilungswechsel und
              Kontotyp noch zusammenpassen. Nicht genutzte oder doppelte Lizenzen werden zurückgebaut.
            </p>
          </article>
        </div>

        <div class="table-wrap decision-table">
          <table>
            <thead>
              <tr>
                <th>Szenario</th>
                <th>Geeignete Basis</th>
                <th>Zusatz prüfen</th>
                <th>Entscheidungskriterium</th>
              </tr>
            </thead>
            <tbody>
              <tr>
                <td>Interner Standardnutzer</td>
                <td>Microsoft 365 F3</td>
                <td>Copilot, Planner</td>
                <td>Regelmäßige Office-, Mail- und Collaboration-Nutzung</td>
              </tr>
              <tr>
                <td>Interner Nutzer mit erhöhtem Schutzbedarf</td>
                <td>Microsoft 365 E5</td>
                <td>Copilot, Visio</td>
                <td>Security-, Compliance- oder erweiterte Analyseanforderungen</td>
              </tr>
              <tr>
                <td>Externer Nutzer</td>
                <td>Entra ID P2, Exchange Online</td>
                <td>Microsoft 365 F3</td>
                <td>Minimaler Zugriff statt Vollpaket, solange fachlich ausreichend</td>
              </tr>
              <tr>
                <td>Service Account</td>
                <td>Keine interaktive Suite</td>
                <td>Defender Add-On</td>
                <td>Technische Notwendigkeit, Schutzbedarf und Auditierbarkeit</td>
              </tr>
            </tbody>
          </table>
        </div>
      </section>
    </main>
  </div>
</template>
