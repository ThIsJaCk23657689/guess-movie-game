<script setup>
import { computed, onMounted, nextTick, ref } from 'vue'

const categories = [
	{ key: 'classic', name: '經典電影', blurb: '時代洗鍊的旋律，記憶中的銀幕片段。' },
	{ key: 'horror', name: '恐怖驚悚片', blurb: '低頻壓迫與心跳共振，誰先破防？' },
	{ key: 'scifi', name: '科幻片', blurb: '合成音色與太空回音，答案在星際深處。' },
	{ key: 'animation', name: '動畫片', blurb: '動畫世界的奇幻冒險，重溫童年時光。' },
	{ key: 'taiwan', name: '國片', blurb: '熟悉的聲線與土地味，猜得到嗎？' },
]

const view = ref('home')
const allGames = ref([])
const loadError = ref('')
const selectedCategory = ref(null)
const roundQuestions = ref([])
const currentIndex = ref(0)

// 回合狀態：idle, playing, answering, revealing, ended
const roundState = ref('idle')
const audioRef = ref(null)

const playDuration = 40
const answerDuration = 5
const timeLeft = ref(playDuration)
const answerTimeLeft = ref(answerDuration)
let playTimer = null
let answerTimer = null

const teams = ref([
	{ id: 1, name: '紅隊', score: 0, chip: 'from-rose-500 to-red-700' },
	{ id: 2, name: '藍隊', score: 0, chip: 'from-sky-500 to-blue-700' },
	{ id: 3, name: '綠隊', score: 0, chip: 'from-emerald-500 to-green-700' },
	{ id: 4, name: '金隊', score: 0, chip: 'from-amber-400 to-yellow-600' },
	{ id: 5, name: '銀隊', score: 0, chip: 'from-slate-300 to-slate-500' },
])

const currentQuestion = computed(() => roundQuestions.value[currentIndex.value])

const playProgress = computed(() => timeLeft.value / playDuration)
const answerProgress = computed(() => answerTimeLeft.value / answerDuration)

const playDisplay = computed(() => Math.ceil(timeLeft.value))
const answerDisplay = computed(() => Math.ceil(answerTimeLeft.value))

const isPlayHurry = computed(() => roundState.value === 'playing' && timeLeft.value <= 10)
const isAnswerHurry = computed(() => roundState.value === 'answering' && answerTimeLeft.value <= 2)

const circleRadius = 42
const circleC = 2 * Math.PI * circleRadius
const playDashOffset = computed(() => circleC * (1 - Math.max(0, Math.min(1, playProgress.value))))
const answerDashOffset = computed(() => circleC * (1 - Math.max(0, Math.min(1, answerProgress.value))))

const TIMER_INTERVAL = 50

const adminForm = ref({
	category: categories[0].name,
	title: '',
	startTime: 0,
	audioFile: null,
	posterFile: null,
})
const adminItems = ref([])

const adminPreview = computed(() =>
	JSON.stringify(
		adminItems.value.map((item) => ({
			id: item.id,
			category: item.category,
			title: item.title,
			audioSrc: item.audioSrc,
			posterSrc: item.posterSrc,
			startTime: item.startTime,
		})),
		null,
		2
	)
)

// 洗牌函式
function shuffle(list) {
	const items = [...list]
	for (let i = items.length - 1; i > 0; i -= 1) {
		const j = Math.floor(Math.random() * (i + 1))
			;[items[i], items[j]] = [items[j], items[i]]
	}
	return items
}

function resetPlayTimer() {
	if (playTimer) clearInterval(playTimer)
	playTimer = null
}

function resetAnswerTimer() {
	if (answerTimer) clearInterval(answerTimer)
	answerTimer = null
}

// 清除所有計時器
function clearTimers() {
	resetPlayTimer()
	resetAnswerTimer()
}

function playAudio(playoffset = 0) {
	const el = audioRef.value
	if (!el) return
	el.currentTime = currentQuestion.value.startTime + playoffset || 0
	el.play()
}

function stopAudio() {
	const el = audioRef.value
	if (!el) return
	el.pause()
	el.currentTime = 0
}

function selectCategory(category) {
	selectedCategory.value = category
	view.value = 'game'
	setupRound()
}

function setupRound() {
	clearTimers()
	stopAudio()
	const pool = allGames.value.filter((game) => game.category === selectedCategory.value.name)
	roundQuestions.value = shuffle(pool).slice(0, 5)
	currentIndex.value = 0
	roundState.value = 'idle'
	timeLeft.value = playDuration
	answerTimeLeft.value = answerDuration
}

function pausePlayTimer() {
	resetPlayTimer()
}

function forceRevealAnswer() {
	revealAnswer(false)
}

async function startPlayback( playoffset = 0 ) {
	if (!currentQuestion.value) return

	resetPlayTimer()
	stopAudio()

	roundState.value = 'playing'
	timeLeft.value = playDuration - playoffset

	try {
		await nextTick()
		await playAudio( playoffset )

		const start = Date.now()

		playTimer = setInterval(() => {
			const elapsed = (Date.now() - start) / 1000
			const remaining = Math.max(0, (playDuration - playoffset) - elapsed)
			timeLeft.value = remaining
			if (remaining <= 0) {
				resetPlayTimer()
				forceRevealAnswer()
			}
		}, TIMER_INTERVAL)
	} catch (err) {
		console.error('Audio playback failed:', err)
		resetPlayTimer()
	}
}

function answerButtonClicked() {
	if (roundState.value === 'playing' )
	{
		// 進入搶答環節
		resetAnswerTimer()
		pausePlayTimer()
		stopAudio()
		roundState.value = 'answering'
		answerTimeLeft.value = answerDuration

		const start = Date.now()
		answerTimer = setInterval(() => {
			const remaining = Math.max(0, answerDuration - (Date.now() - start) / 1000)
			answerTimeLeft.value = remaining
			if (remaining <= 0) {
				resetAnswerTimer()
				roundState.value = 'playing'
				startPlayback(playDuration - timeLeft.value)
			}
		}, TIMER_INTERVAL)
	}
	else
	{
		// 答錯，回到等待狀態
		resetAnswerTimer()
		roundState.value = 'playing'
		startPlayback(playDuration - timeLeft.value)
	}
}

function revealAnswer(playAudioOnReveal = true) {
	clearTimers()
	roundState.value = 'revealing'
	if (playAudioOnReveal) {
		playAudio(playDuration - timeLeft.value)
	}
}
function nextQuestion() {
	clearTimers()
	stopAudio()
	if (currentIndex.value < roundQuestions.value.length - 1) {
		currentIndex.value += 1
		roundState.value = 'idle'
		timeLeft.value = playDuration
		answerTimeLeft.value = answerDuration
	} else {
		roundState.value = 'ended'
	}
}

function adjustScore(team, delta) {
	team.score += delta
}

function handleFile(event, field) {
	const file = event.target.files?.[0] ?? null
	adminForm.value[field] = file
}

function addAdminItem() {
	if (!adminForm.value.title) return
	const audioSrc = adminForm.value.audioFile
		? `audio/${adminForm.value.audioFile.name}`
		: ''
	const posterSrc = adminForm.value.posterFile
		? `posters/${adminForm.value.posterFile.name}`
		: ''
	adminItems.value.push({
		id: Date.now(),
		category: adminForm.value.category,
		title: adminForm.value.title,
		startTime: Number(adminForm.value.startTime) || 0,
		audioSrc,
		posterSrc,
	})
	adminForm.value.title = ''
	adminForm.value.startTime = 0
	adminForm.value.audioFile = null
	adminForm.value.posterFile = null
}

onMounted(async () => {
	try {
		const res = await fetch(`${import.meta.env.BASE_URL}games.json`)
		if (!res.ok) throw new Error('Failed to load games.json')
		allGames.value = await res.json()
	} catch (err) {
		loadError.value = err.message || 'Failed to load game data'
	}
})
</script>

<template>
	<div class="min-h-screen bg-slate-950 text-slate-100">
		<header class="sticky top-0 z-20 border-b border-slate-800/60 bg-slate-950/80 backdrop-blur">
			<div class="mx-auto flex max-w-6xl items-center justify-between px-6 py-4">
				<div>
					<p class="text-xs uppercase tracking-[0.4em] text-amber-300/80">Movie Music Rush</p>
					<h1 class="text-2xl font-bold text-amber-100">影音先瘋</h1>
				</div>
				<div class="flex items-center gap-3 text-sm">
					<button
						class="rounded-full border border-amber-200/40 px-4 py-2 text-amber-100 hover:bg-amber-200/10"
						@click="view = 'home'">
						首頁
					</button>
					<button class="rounded-full border border-slate-700 px-4 py-2 text-slate-200 hover:bg-slate-800"
						@click="view = 'admin'">
						後台管理
					</button>
				</div>
			</div>
		</header>

		<main class="mx-auto max-w-6xl px-6 pb-36 pt-10">
			<section v-if="view === 'home'">
				<div class="mb-10 grid gap-6 text-left md:grid-cols-2">
					<div class="rounded-3xl border border-slate-800/70 bg-slate-900/60 p-6">
						<h2 class="text-3xl font-semibold text-amber-100">選擇主題，讓節奏決勝負</h2>
						<p class="mt-3 text-slate-300">
							每回合從分類中抽出 5 題，播放 40 秒音樂後搶答。
						</p>
						<p v-if="loadError" class="mt-3 text-sm text-rose-300">
							{{ loadError }}
						</p>
					</div>
					<div class="rounded-3xl border border-slate-800/70 bg-slate-900/60 p-6">
						<h3 class="text-lg font-semibold text-slate-100">快速規則</h3>
						<ul class="mt-3 space-y-2 text-sm text-slate-300">
							<li>每題開始會播放 40 秒 的音樂片段。</li>
							<li>各組代表的猜賽者必須在 40 秒內進行按鈴搶答，最快按鈴者獲得發言權。</li>
							<li>取得發言權後，必須在 5 秒內 說出正確電影名稱。</li>
						</ul>
					</div>
				</div>

				<div class="grid gap-6 md:grid-cols-2 lg:grid-cols-3">
					<button v-for="category in categories" :key="category.key"
						class="group rounded-[28px] border border-slate-800/70 bg-gradient-to-br from-slate-900/80 via-slate-900/30 to-slate-950 p-6 text-left transition hover:-translate-y-1 hover:border-amber-200/60"
						@click="selectCategory(category)">
						<div class="flex items-center justify-between">
							<h3 class="text-2xl font-semibold text-amber-100">{{ category.name }}</h3>
							<span class="text-xs uppercase tracking-[0.3em] text-slate-500 group-hover:text-amber-300">
								Theme
							</span>
						</div>
						<p class="mt-4 text-sm text-slate-300">{{ category.blurb }}</p>
						<div class="mt-6 text-sm text-amber-200/80">進入搶答 →</div>
					</button>
				</div>
			</section>

			<section v-else-if="view === 'game'">
				<div class="mb-6 flex flex-wrap items-center justify-between gap-4">
					<div>
						<p class="text-xs uppercase tracking-[0.4em] text-amber-300/70">Round</p>
						<h2 class="text-3xl font-semibold text-amber-100">
							{{ selectedCategory?.name }} 主題
						</h2>
						<p class="mt-2 text-sm text-slate-300">
							第 {{ currentIndex + 1 }} 題 / 共 {{ roundQuestions.length }} 題
						</p>
					</div>
					<div class="flex items-center gap-3">
						<button
							class="rounded-full border border-slate-700 px-5 py-2 text-sm text-slate-200 hover:bg-slate-800"
							@click="setupRound">
							重新抽題
						</button>
						<button
							class="rounded-full border border-amber-200/40 px-5 py-2 text-sm text-amber-100 hover:bg-amber-200/10"
							@click="view = 'home'">
							返回首頁
						</button>
					</div>
				</div>

				<div v-if="!currentQuestion"
					class="rounded-3xl border border-slate-800/70 bg-slate-900/60 p-8 text-center">
					<p class="text-slate-300">此分類題目不足，請在後台新增題庫。</p>
				</div>

				<div v-else class="grid gap-6 lg:grid-cols-1">
					<div class="rounded-3xl border border-slate-800/70 bg-slate-900/60 p-6">
						<div class="flex items-start justify-between gap-4">
							<div>
								<p class="text-xs uppercase tracking-[0.4em] text-slate-500">Now Playing</p>
								<h3 class="mt-2 text-2xl font-semibold text-slate-100">
									第 {{ currentIndex + 1 }} 關
								</h3>
								<p class="mt-1 text-sm text-slate-400">
									從 {{ currentQuestion.startTime }} 秒開始播放
								</p>
							</div>
							<div class="flex items-center gap-4">
								<div class="relative h-24 w-24">
									<svg class="h-24 w-24 -rotate-90" viewBox="0 0 100 100">
										<circle cx="50" cy="50" r="42" stroke="rgba(148,163,184,0.2)" stroke-width="8"
											fill="none" />
										<circle cx="50" cy="50" r="42" stroke="rgba(251,191,36,0.9)" stroke-width="8"
											stroke-linecap="round" fill="none"
											:style="{ strokeDasharray: circleC, strokeDashoffset: playDashOffset }" />
									</svg>
									<div class="absolute inset-0 flex items-center justify-center text-2xl font-semibold"
										:class="[isPlayHurry ? 'text-rose-400 shake' : 'text-amber-100']">
										{{ playDisplay }}
									</div>
								</div>
								<div class="relative h-24 w-24">
									<svg class="h-24 w-24 -rotate-90" viewBox="0 0 100 100">
										<circle cx="50" cy="50" r="42" stroke="rgba(148,163,184,0.2)" stroke-width="8"
											fill="none" />
										<circle cx="50" cy="50" r="42" stroke="rgba(248,113,113,0.9)" stroke-width="8"
											stroke-linecap="round" fill="none"
											:style="{ strokeDasharray: circleC, strokeDashoffset: answerDashOffset }" />
									</svg>
									<div class="absolute inset-0 flex items-center justify-center text-2xl font-semibold"
										:class="[isAnswerHurry ? 'text-rose-400 shake' : 'text-rose-200']">
										{{ answerDisplay }}
									</div>
								</div>
							</div>
						</div>

						<div class="mt-6 flex flex-wrap gap-3">
							<button
								class="rounded-full bg-amber-400 px-6 py-2 text-sm font-semibold text-slate-900 hover:bg-amber-300 disabled:cursor-not-allowed disabled:bg-slate-600"
								:disabled="roundState !== 'idle'" @click="startPlayback(0)">
								開始播放
							</button>
							<button
								class="rounded-full border border-fuchsia-300/60 px-6 py-2 text-sm font-semibold text-fuchsia-100 hover:bg-fuchsia-500/20 disabled:cursor-not-allowed disabled:bg-slate-600 disabled:text-slate-900 disabled:border-none"
								:disabled="roundState !== 'playing' && roundState !== 'answering'" @click="answerButtonClicked">
								{{ roundState === 'answering' ? '答錯！' : '搶答' }}
							</button>
							<button
								class="rounded-full border border-rose-300/60 px-6 py-2 text-sm font-semibold text-rose-100 hover:bg-rose-500/20 disabled:cursor-not-allowed disabled:bg-slate-600 disabled:text-slate-900 disabled:border-none"
								:disabled="roundState !== 'answering'" @click="revealAnswer(true)">
								顯示答案
							</button>
							<button
								class="rounded-full border border-slate-700 px-6 py-2 text-sm text-slate-200 hover:bg-slate-800 disabled:cursor-not-allowed disabled:bg-slate-600 disabled:text-slate-900"
								:disabled="roundState !== 'revealing'" @click="nextQuestion">
								下一題
							</button>
						</div>

					</div>
				</div>
				<audio ref="audioRef" :src="currentQuestion?.audioSrc"></audio>

				<div v-if="roundState === 'ended'"
					class="mt-4 rounded-3xl border border-slate-800/70 bg-slate-900/60 p-8 text-center">
					<p class="text-slate-300">回合結束！可以重新抽題或切換主題。</p>
				</div>

				<!-- Modal 彈跳視窗 - 顯示答案 -->
				<transition name="modal-fade">
					<div v-if="roundState === 'revealing'"
						class="fixed inset-0 z-50 flex items-center justify-center bg-black/80 backdrop-blur-sm"
						@click.self="nextQuestion">
						<div class="relative max-h-[90vh] max-w-4xl w-full mx-4 overflow-hidden rounded-3xl border border-amber-200/40 bg-slate-900 shadow-2xl">
							<!-- 海報圖片 -->
							<div class="flex items-center justify-center bg-slate-950 p-4">
								<img :src="currentQuestion.posterSrc" alt="Movie Poster"
									class="max-h-[60vh] w-auto object-contain rounded-xl shadow-lg" />
							</div>

							<!-- 電影名稱 -->
							<div class="px-4 pt-2 text-center bg-gradient-to-b from-slate-900 to-slate-950">
								<!-- 主標題：台灣名稱和英文名稱 -->
								<div class="space-y-1">
									<h2 class="text-4xl font-bold text-amber-100">
										{{ currentQuestion.title_tw }}
									</h2>
									<p class="text-2xl font-semibold text-amber-200/80">
										{{ currentQuestion.title_en }}
									</p>
								</div>

								<!-- 次標題：中國和香港名稱（如果有） -->
								<div v-if="currentQuestion.title_cn || currentQuestion.title_hk" class="mt-2 space-y-1">
									<p v-if="currentQuestion.title_cn" class="text-sm text-slate-400">
										中國翻譯：{{ currentQuestion.title_cn }}
									</p>
									<p v-if="currentQuestion.title_hk" class="text-sm text-slate-400">
										香港翻譯：{{ currentQuestion.title_hk }}
									</p>
								</div>

								<!-- 關閉提示 -->
								<div class="my-4">
									<button
										class="rounded-full bg-amber-400 px-8 py-3 text-sm font-semibold text-slate-900 hover:bg-amber-300"
										@click="nextQuestion">
										下一題
									</button>
								</div>
							</div>
						</div>
					</div>
				</transition>
			</section>

			<section v-else-if="view === 'admin'">
				<div class="mb-8 grid gap-6 lg:grid-cols-[1.1fr_0.9fr]">
					<div class="rounded-3xl border border-slate-800/70 bg-slate-900/60 p-6">
						<h2 class="text-2xl font-semibold text-amber-100">後台管理</h2>
						<p class="mt-2 text-sm text-slate-300">
							選擇本地檔案、填入起始秒數與電影名稱，即可產生 JSON。
						</p>
						<div class="mt-6 grid gap-4">
							<label class="text-sm text-slate-300">
								分類
								<select v-model="adminForm.category"
									class="mt-2 w-full rounded-xl border border-slate-700 bg-slate-950/80 px-4 py-2 text-slate-100">
									<option v-for="category in categories" :key="category.key" :value="category.name">
										{{ category.name }}
									</option>
								</select>
							</label>
							<label class="text-sm text-slate-300">
								電影名稱
								<input v-model="adminForm.title" type="text" placeholder="輸入電影名稱"
									class="mt-2 w-full rounded-xl border border-slate-700 bg-slate-950/80 px-4 py-2 text-slate-100" />
							</label>
							<label class="text-sm text-slate-300">
								開始秒數
								<input v-model.number="adminForm.startTime" type="number" min="0"
									class="mt-2 w-full rounded-xl border border-slate-700 bg-slate-950/80 px-4 py-2 text-slate-100" />
							</label>
							<label class="text-sm text-slate-300">
								音樂檔案
								<input type="file" accept="audio/*"
									class="mt-2 w-full rounded-xl border border-slate-700 bg-slate-950/80 px-4 py-2 text-slate-100 file:mr-4 file:rounded-lg file:border-0 file:bg-amber-400 file:px-4 file:py-2 file:text-sm file:font-semibold file:text-slate-900"
									@change="(event) => handleFile(event, 'audioFile')" />
							</label>
							<label class="text-sm text-slate-300">
								海報檔案
								<input type="file" accept="image/*"
									class="mt-2 w-full rounded-xl border border-slate-700 bg-slate-950/80 px-4 py-2 text-slate-100 file:mr-4 file:rounded-lg file:border-0 file:bg-amber-400 file:px-4 file:py-2 file:text-sm file:font-semibold file:text-slate-900"
									@change="(event) => handleFile(event, 'posterFile')" />
							</label>
							<button
								class="mt-2 w-full rounded-full bg-amber-400 px-6 py-3 text-sm font-semibold text-slate-900 hover:bg-amber-300"
								@click="addAdminItem">
								加入 JSON 清單
							</button>
						</div>
					</div>

					<div class="rounded-3xl border border-slate-800/70 bg-slate-900/60 p-6">
						<h3 class="text-lg font-semibold text-slate-100">JSON 預覽</h3>
						<p class="mt-2 text-xs text-slate-400">將檔案放進 public/audio 與 public/posters</p>
						<div
							class="mt-4 rounded-2xl border border-slate-800 bg-slate-950/70 p-4 text-xs text-amber-100">
							<pre class="whitespace-pre-wrap">{{ adminPreview }}</pre>
						</div>
					</div>
				</div>
			</section>
		</main>

		<footer class="fixed bottom-0 left-0 right-0 z-30 border-t border-slate-800/70 bg-slate-950/90 backdrop-blur">
			<div class="mx-auto flex max-w-6xl flex-wrap gap-3 px-6 py-4">
				<div class="text-xs uppercase tracking-[0.4em] text-slate-500">Scoreboard</div>
				<div class="flex flex-1 flex-wrap gap-3">
					<div v-for="team in teams" :key="team.id"
						class="flex items-center gap-3 rounded-2xl border border-slate-800/70 bg-slate-900/70 px-4 py-2 text-sm text-slate-100">
						<div class="flex items-center gap-2">
							<span class="h-2 w-2 rounded-full bg-gradient-to-r" :class="team.chip"></span>
							<button class="font-semibold text-slate-100 hover:text-amber-200">
								{{ team.name }}
							</button>
						</div>
						<div class="text-lg font-semibold text-amber-100">{{ team.score }}</div>
						<div class="flex gap-2">
							<button
								class="rounded-full border border-slate-700 px-2 py-1 text-xs text-slate-200 hover:bg-slate-800"
								@click="adjustScore(team, -1)">
								-1
							</button>
							<button
								class="rounded-full border border-slate-700 px-2 py-1 text-xs text-slate-200 hover:bg-slate-800"
								@click="adjustScore(team, 1)">
								+1
							</button>
						</div>
					</div>
				</div>
			</div>
		</footer>
	</div>
</template>

<style scoped>
/* Modal 淡入淡出動畫 */
.modal-fade-enter-active,
.modal-fade-leave-active {
	transition: opacity 0.3s ease;
}

.modal-fade-enter-from,
.modal-fade-leave-to {
	opacity: 0;
}

.modal-fade-enter-active .relative {
	animation: modal-scale-in 0.3s ease-out;
}

.modal-fade-leave-active .relative {
	animation: modal-scale-out 0.3s ease-in;
}

@keyframes modal-scale-in {
	from {
		transform: scale(0.9);
		opacity: 0;
	}
	to {
		transform: scale(1);
		opacity: 1;
	}
}

@keyframes modal-scale-out {
	from {
		transform: scale(1);
		opacity: 1;
	}
	to {
		transform: scale(0.9);
		opacity: 0;
	}
}
</style>
