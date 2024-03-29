<template>
	<div>
		<h1>CrossCheck Done Docs</h1>
		<div class="form-group my-3">
			<input v-model="id" class="form-control" placeholder="Enter id of the doc" type="text">
		</div>
		<button :disabled="!id" class="btn-lg btn-primary" @click="submit">CrossCheck Now</button>
		<p v-if="error" class="my-3 text-danger">{{ error }}</p>
		<hr class="my-5">
		<div v-if="loading">
			<p class="text-center">Loading</p>
		</div>
		<div v-else-if="fetched" class="mb-5">
			<h1 style="text-transform: capitalize;">{{ name }}</h1>
			<h2>Non Specials - Total:{{ nonSpecials.length }}</h2>
			<div class="mb-5">
				<div v-for="(question, index) in nonSpecials" :key="`nonSpecial`+index">
					<p>Order: {{ question.order }}</p>
					<p>{{ question.question }}</p>
					<p>A: {{ question.a }}</p>
					<p>B: {{ question.b }}</p>
					<p>C: {{ question.c }}</p>
					<p>D: {{ question.d }}</p>
					<p v-if="question.e">E: {{ question.e }}</p>
					<hr class="my-2">
				</div>

				<div class="my-3">
					<button class="btn-lg btn-primary" @click="uploadQuestions">Upload Questions</button>
				</div>
			</div>
			<hr class="my-5">
			<h2>Specials - Total:{{ specials.length }}</h2>
			<p v-for="(question, index) in specials" :key="`special`+index">
				{{ question }}
			</p>
		</div>
	</div>
</template>

<script lang="ts">
import { defineComponent, ref } from '@vue/composition-api'
import { getName, parseContent } from '../../scripts/parseContent'
import { saveQuestionToDatabase } from '@/utils/firebase'

export default defineComponent({
	setup () {
		const id = ref('')
		const name = ref('')
		const fetched = ref(false)
		const error = ref('')
		const loading = ref(false)
		const specials = ref([] as any[])
		const nonSpecials = ref([] as any[])
		const submit = async () => {
			try {
				error.value = ''
				const link = `https://docs.google.com/document/d/${id.value}/export?format=txt`
				const res = await fetch(link)
				const n = JSON.parse(res.headers.get('content-disposition')?.split(';')[1].split('=')[1] ?? '').split('.')[0] ?? ''
				name.value = n.trim()
				const { subject, year, examType } = getName(name.value)
				const content = await res.text()
				fetched.value = true

				const { special, nonSpecial } = parseContent(content, link, subject, examType, year)
				specials.value = special
				nonSpecials.value = nonSpecial
			} catch (e: any) {
				error.value = e.message
				specials.value = []
				nonSpecials.value = []
			}
		}
		const uploadQuestions = async () => {
			loading.value = true
			try {
				for (const question of nonSpecials.value) {
					await saveQuestionToDatabase(question, 'objectives', true)
						.catch((e) => {
							if (!e.message.startsWith('Missing')) throw e
						})
				}
				alert('All Questions Uploaded Successfully')
			} catch (e: any) {
				alert('Error: ' + e.message)
			}
			loading.value = false
		}

		return {
			id, error, submit, specials, nonSpecials, fetched, name, uploadQuestions,
			loading
		}
	}
})
</script>
