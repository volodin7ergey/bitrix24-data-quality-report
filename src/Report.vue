<template>
	<div class="app-start-box" v-if="isAppLoading">
		<v-progress-circular indeterminate />
	</div>

	<template v-else>
		<div class="filters-box">
			<div style="display: flex; gap: 1rem">


				<v-checkbox
					label="Все сущности"
					value="ALL"
					v-model="entityTypes"
					@update:modelValue="toggleAll"
				/>

				<v-checkbox
					label="Сделки"
					value="DEAL"
					v-model="entityTypes"
					:disabled="entityTypes.includes('ALL')"
				/>

				<v-checkbox
					label="Компании"
					value="COMPANY"
					v-model="entityTypes"
					:disabled="entityTypes.includes('ALL')"
				/>

				<v-checkbox
					label="Контакты"
					value="CONTACT"
					v-model="entityTypes"
					:disabled="entityTypes.includes('ALL')"
				/>

								<v-date-input
					v-model="dateFrom"
					label="Период с"
					variant="outlined"
					density="compact"
					max-width="200"
					autocomplete="off"
					clearable
				/>

				<v-date-input
					v-model="dateTo"
					label="по"
					variant="outlined"
					density="compact"
					max-width="200"
					autocomplete="off"
					clearable
				/>
			</div>

			<v-btn
				color="primary"
				@click="onLoadReport"
				:disabled="!entityTypes.length || isLoading"
				:loading="isLoading"
				>Сформировать отчёт
			</v-btn>
		</div>

		<div v-if="rows.length" class="kpi-grid">
	<v-card class="kpi-card">
		<div class="kpi-title">Проблем найдено</div>
		<div class="kpi-value red--text">{{ problemsCount }}</div>
	</v-card>

	<v-card class="kpi-card" >
		<div class="kpi-title">Сделки</div>
		<div class="kpi-value blue--text">{{ byType.deal }}</div>
	</v-card>

	<v-card class="kpi-card">
		<div class="kpi-title">Компании</div>
		<div class="kpi-value purple--text">{{ byType.company }}</div>
	</v-card>

	<v-card class="kpi-card">
		<div class="kpi-title">Контакты</div>
		<div class="kpi-value green--text">{{ byType.contact }}</div>
	</v-card>
</div>
		<div class="content-box">
			<v-data-table
				v-if="rows.length"
				:headers="headers"
				:items="rows"
				item-value="key"
				class="elevation-1 rounded-lg"
				density="comfortable"
				:items-per-page="10"
			>
				<!-- Тип -->
				<template #item.type="{ item }">
					<v-chip
						:color="
							{
								Сделка: 'blue',
								Компания: 'purple',
								Контакт: 'green',
							}[item.type]
						"
						size="small"
					>
						{{ item.type }}
					</v-chip>
				</template>

				<!-- Название -->
				<template #item.title="{ item }">
					<a :href="item.link" target="_blank">
						{{ item.title }}
					</a>
				</template>

				<!-- Проблемы -->
				<template #item.problems="{ item }">
					<v-chip
						v-for="p in item.problems"
						:key="p"
						color="pink"
						size="small"
						class="mr-1 mb-1"
					>
						{{ p }}
					</v-chip>
				</template>
			</v-data-table>
			<!-- Сообщение об отсутствии проблем -->
			<v-alert
				v-if="!rows.length && loaded"
				type="info"
				class="mt-6"
				border="start"
			>
				Проблемных сущностей не найдено
			</v-alert>
		</div>
	</template>
</template>

<script setup lang="ts">
import { onMounted, ref, watch, computed } from 'vue'

type Deal = {
	ID: string
	TITLE: string
	CONTACT_ID: string | null
	COMPANY_ID: string | null
	DATE_CREATE: string
}

type Company = {
	ID: string
	TITLE: string
	PHONE?: any[]
	EMAIL?: any[]
	ADDRESS?: string
	DATE_CREATE: string
}

type Contact = {
	ID: string
	NAME: string
	LAST_NAME: string
	PHONE?: any[]
	EMAIL?: any[]
	ADDRESS?: string
	DATE_CREATE: string
}

type ProblemRow = {
	key: string
	type: string
	title: string
	link: string
	problems: string[]
}
const headers = [
	{ title: 'Тип сущности', key: 'type' },
	{ title: 'Элемент', key: 'title' },
	{
		title: 'Проблемы',
		key: 'problems',
		sortable: true,
		sort: (a: any, b: any) => a.length - b.length,
	},
]

const isAppLoading = ref(true)

const rows = ref<ProblemRow[]>([])

const dateFrom = ref('')
const dateTo = ref('')

const loaded = ref(false)

const entityTypes = ref<string[]>(['ALL'])

function toggleAll() {
	if (entityTypes.value.includes('ALL')) entityTypes.value = ['ALL']
}

function loadRequisites(
	entityTypeId: number,
	entityIds: string[],
): Promise<any[]> {
	return new Promise(resolve => {
		BX24.callMethod(
			'crm.requisite.list',
			{
				filter: {
					ENTITY_TYPE_ID: entityTypeId,
					ENTITY_ID: entityIds,
				},
			},
			(res: any) => resolve(res.data()),
		)
	})
}

function loadAddresses(requisiteIds: string[]): Promise<any[]> {
	return new Promise(resolve => {
		BX24.callMethod(
			'crm.address.list',
			{
				filter: {
					ENTITY_TYPE_ID: 8,
					ENTITY_ID: requisiteIds,
				},
			},
			(res: any) => resolve(res.data()),
		)
	})
}

function hasValidAddress(addresses: any[]) {
	return addresses.some(a => {
		return [a.ADDRESS_1, a.ADDRESS_2, a.CITY, a.REGION, a.PROVINCE].some(
			v => v && v.toString().trim().length > 2,
		)
	})
}
function buildFilter() {
	const filter: any = {}

	if (dateFrom.value) filter['>=DATE_CREATE'] = dateFrom.value
	if (dateTo.value) filter['<=DATE_CREATE'] = dateTo.value

	return filter
}

function checkDeal(deal: Deal, domain: string) {
	const problems: string[] = []

	if (!deal.CONTACT_ID) problems.push('Нет контакта')
	if (!deal.COMPANY_ID) problems.push('Нет компании')

	if (!problems.length) return null

	return {
		key: 'deal' + deal.ID,
		type: 'Сделка',
		title: deal.TITLE,
		link: `https://${domain}/crm/deal/details/${deal.ID}/`,
		problems,
	}
}

function checkCompany(company: Company, domain: string) {
	const problems: string[] = []

	if (!company.ADDRESS) problems.push('Нет адреса')
	if (!company.PHONE?.length) problems.push('Нет телефона')

	if (!problems.length) return null

	return {
		key: 'company' + company.ID,
		type: 'Компания',
		title: company.TITLE,
		link: `https://${domain}/crm/company/details/${company.ID}/`,
		problems,
	}
}

function checkContact(contact: Contact, domain: string) {
	const problems: string[] = []

	if (!contact.ADDRESS) problems.push('Нет адреса')
	if (!contact.PHONE?.length) problems.push('Нет телефона')
	if (!contact.EMAIL?.length) problems.push('Нет email')

	if (!problems.length) return null

	return {
		key: 'contact' + contact.ID,
		type: 'Контакт',
		title: `${contact.NAME} ${contact.LAST_NAME}`,
		link: `https://${domain}/crm/contact/details/${contact.ID}/`,
		problems,
	}
}

function loadDeals(filter: any): Promise<Deal[]> {
	return new Promise(resolve => {
		BX24.callMethod(
			'crm.deal.list',
			{
				select: ['ID', 'TITLE', 'CONTACT_ID', 'COMPANY_ID', 'DATE_CREATE'],
				filter,
			},
			(res: any) => resolve(res.data()),
		)
	})
}

function loadCompanies(filter: any): Promise<Company[]> {
	return new Promise(resolve => {
		BX24.callMethod(
			'crm.company.list',
			{
				select: ['ID', 'TITLE', 'PHONE', 'ADDRESS', 'DATE_CREATE'],
				filter,
			},
			(res: any) => resolve(res.data()),
		)
	})
}

function loadContacts(filter: any): Promise<Contact[]> {
	return new Promise(resolve => {
		BX24.callMethod(
			'crm.contact.list',
			{
				select: [
					'ID',
					'NAME',
					'LAST_NAME',
					'PHONE',
					'EMAIL',
					'ADDRESS',
					'ADDRESS_2',
					'ADDRESS_CITY',
					'DATE_CREATE',
				],
				filter,
			},
			(res: any) => resolve(res.data()),
		)
	})
}

const total = computed(() => rows.value.length)

const problemsCount = computed(() => rows.value.length)

const percent = computed(() => {
	if (!total.value) return 0
	return Math.round((problemsCount.value / total.value) * 100)
})

const byType = computed(() => {
	return {
		deal: rows.value.filter(r => r.type === 'Сделка').length,
		company: rows.value.filter(r => r.type === 'Компания').length,
		contact: rows.value.filter(r => r.type === 'Контакт').length,
	}
})
async function loadReport() {
	rows.value = []

	const result = []
	loaded.value = false

	const domain = BX24.getDomain()

	const filter = buildFilter()

	const types = entityTypes.value.includes('ALL')
		? ['DEAL', 'COMPANY', 'CONTACT']
		: entityTypes.value

	if (types.includes('DEAL')) {
		const deals = await loadDeals(filter)

		for (const deal of deals) {
			const row = checkDeal(deal, domain)

			if (row) result.push(row)
		}
	}

	function hasValidRequisites(requisites: any[]): boolean {
		return requisites.some(r => {
			// игнорируем реквизиты, созданные только для адреса
			if (r.ADDRESS_ONLY === 'Y') return false

			// проверяем реальные ключевые поля
			const inn = r.INN?.toString().trim()
			const kpp = r.KPP?.toString().trim()
			const bank = r.BANK?.toString().trim()
			const bankAccount = r.BANK_ACCOUNT?.toString().trim()
			const name = r.NAME?.toString().trim()

			return inn || kpp || bank || bankAccount || name
		})
	}

	if (types.includes('COMPANY')) {
		const companies = await loadCompanies(filter)
		const companyIds = companies.map(c => c.ID)

		// 🔹 Реквизиты
		const requisites = await loadRequisites(4, companyIds)

		console.log(requisites)
		const requisitesByCompany: Record<string, any[]> = {}
		requisites.forEach(r => {
			if (!requisitesByCompany[r.ENTITY_ID])
				requisitesByCompany[r.ENTITY_ID] = []
			requisitesByCompany[r.ENTITY_ID].push(r)
		})

		const requisiteIds = requisites.map(r => r.ID)
		const addresses = await loadAddresses(requisiteIds)

		console.log(addresses)
		const addressesByRequisite: Record<string, any[]> = {}
		addresses.forEach(a => {
			if (!addressesByRequisite[a.ENTITY_ID])
				addressesByRequisite[a.ENTITY_ID] = []
			addressesByRequisite[a.ENTITY_ID].push(a)
		})

		// 🔹 Контакты по каждой компании
		const contactsByCompany: Record<string, any[]> = {}
		for (const companyId of companyIds) {
			const contacts = await loadContacts({ COMPANY_ID: companyId })
			contactsByCompany[companyId] = contacts
		}

		// 🔹 Проверка компаний
		for (const company of companies) {
			const problems: string[] = []
			const companyRequisites = requisitesByCompany[company.ID] || []

			// Проверка реквизитов (ключевые поля, игнорируем ADDRESS_ONLY)
			if (!hasValidRequisites(companyRequisites))
				problems.push('Нет реквизитов')

			// Проверка адреса через реквизиты
			let validAddress = false
			for (const req of companyRequisites) {
				const addr = addressesByRequisite[req.ID] || []
				if (hasValidAddress(addr)) {
					validAddress = true
					break
				}
			}
			if (!validAddress) problems.push('Нет адреса')

			// Телефон
			if (!company.PHONE?.length) problems.push('Нет телефона')

			// Привязанные контакты
			if (!contactsByCompany[company.ID]?.length)
				problems.push('Нет привязанного контакта')

			// Если есть проблемы — добавляем в результат
			if (problems.length) {
				result.push({
					key: 'company' + company.ID,
					type: 'Компания',
					title: company.TITLE,
					link: `https://${domain}/crm/company/details/${company.ID}/`,
					problems,
				})
			}
		}
	}

	if (types.includes('CONTACT')) {
		const contacts = await loadContacts(filter)
		const contactIds = contacts.map(c => c.ID)

		// 🔹 Реквизиты и адреса
		const requisites = await loadRequisites(3, contactIds)
		const requisitesByContact: Record<string, any[]> = {}
		requisites.forEach(r => {
			if (!requisitesByContact[r.ENTITY_ID])
				requisitesByContact[r.ENTITY_ID] = []
			requisitesByContact[r.ENTITY_ID].push(r)
		})

		const requisiteIds = requisites.map(r => r.ID)
		const addresses = await loadAddresses(requisiteIds)
		const addressesByRequisite: Record<string, any[]> = {}
		addresses.forEach(a => {
			if (!addressesByRequisite[a.ENTITY_ID])
				addressesByRequisite[a.ENTITY_ID] = []
			addressesByRequisite[a.ENTITY_ID].push(a)
		})

		// 🔹 Проверка контактов
		for (const contact of contacts) {
			const problems: string[] = []
			const contactRequisites = requisitesByContact[contact.ID] || []

			// Реквизиты
			if (!hasValidRequisites(contactRequisites))
				problems.push('Нет реквизитов')

			// Адрес
			let validAddress = false
			for (const req of contactRequisites) {
				const addr = addressesByRequisite[req.ID] || []
				if (hasValidAddress(addr)) {
					validAddress = true
					break
				}
			}
			if (!validAddress) problems.push('Нет адреса')

			// Телефон
			if (!contact.PHONE?.length) problems.push('Нет телефона')

			// Email
			if (!contact.EMAIL?.length) problems.push('Нет email')

			if (problems.length) {
				result.push({
					key: 'contact' + contact.ID,
					type: 'Контакт',
					title: `${contact.NAME} ${contact.LAST_NAME}`,
					link: `https://${domain}/crm/contact/details/${contact.ID}/`,
					problems,
				})
			}
		}
	}

	rows.value = result
	loaded.value = true
}

const isLoading = ref(false)

const onLoadReport = async () => {
	try {
		isLoading.value = true

		await loadReport()
	} catch (e: unknown) {
		console.log(e)
	} finally {
		isLoading.value = false
	}
}

const sleep = (ms: number) => new Promise(res => setTimeout(res, ms))

onMounted(async () => {
	await sleep(1000)

	isAppLoading.value = false
})
</script>

<style scoped>
.app-start-box {
	width: 100%;
	height: 100dvh;
	display: grid;
	place-items: center;
}

.filters-box {
	margin-bottom: 1rem;
}

.report {
	padding: 20px;
	font-family: Arial;
}

.filters {
	margin-bottom: 20px;
	display: flex;
	gap: 20px;
	flex-wrap: wrap;
}

.entity-filter {
	display: flex;
	gap: 10px;
}

table {
	border-collapse: collapse;
	width: 100%;
}

th,
td {
	border: 1px solid #ccc;
	padding: 8px;
}

th {
	background: #f3f3f3;
}

a {
	color: #2a6edc;
	text-decoration: none;
}

a:hover {
	text-decoration: underline;
}

.my-date-input ::v-deep(.v-date-picker-calendar__day) {
	width: 30px;
	height: 30px;
	font-size: 0.8rem;
	margin: 1px;
}

/* Стили для выбранного дня */
.my-date-input ::v-deep(.v-date-picker-calendar__day--selected) {
	background-color: #2a6edc;
	color: white;
	border-radius: 4px;
}

.my-date-input ::v-deep(.v-date-picker) {
	padding: 4px;
}

.my-date-input ::v-deep(.v-date-picker-calendar__day) {
	width: 28px;
	height: 28px;
	font-size: 0.75rem;
	margin: 1px;
}

.v-data-table {
	border-radius: 12px;
}

.v-data-table :deep(tbody tr:hover) {
	background: #f5f7fa;
	cursor: pointer;
}

.kpi-grid {
	display: grid;
	grid-template-columns: repeat(auto-fit, minmax(160px, 1fr));
	gap: 16px;
	margin-bottom: 20px;
}

.kpi-card {
	padding: 16px;
	border-radius: 12px;
	transition: transform 0.2s, box-shadow 0.2s;
	box-shadow: 0 2px 6px rgba(0,0,0,0.1);
}

.kpi-card:hover {
	transform: translateY(-4px);
	box-shadow: 0 4px 12px rgba(0,0,0,0.15);
}
.kpi-title {
	font-size: 0.85rem;
	color: #777;
}

.kpi-value {
	font-size: 1.6rem;
	font-weight: 600;
}
</style>
