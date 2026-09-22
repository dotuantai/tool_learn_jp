<script setup lang="ts">
import { computed, onMounted, onUnmounted, ref } from 'vue'
import { katakanaData, type KatakanaCharacter } from '@/data/katakana'
import { speakJapaneseWord } from '@/utils/speakJapanese'

// ── Types ──────────────────────────────────────────────────────────────────
type QuizMode = 'char-to-romaji' | 'romaji-to-char' | 'random'
type WordRange = 'all' | 'first15' | 'mid15' | 'last16'
type WordOrder = 'ordered' | 'shuffle'
type AnswerState = 'idle' | 'correct' | 'incorrect'

interface KataQuestion {
  item: KatakanaCharacter
  mode: 'char-to-romaji' | 'romaji-to-char'
  options: string[]
  correct: string
  isReview: boolean
}

// ── Helpers ─────────────────────────────────────────────────────────────────
function shuffle<T>(arr: T[]): T[] {
  const result = [...arr]
  for (let i = result.length - 1; i > 0; i--) {
    const j = Math.floor(Math.random() * (i + 1))
    const temp = result[i]!
    result[i] = result[j]!
    result[j] = temp
  }
  return result
}

function getRangeData(range: WordRange): KatakanaCharacter[] {
  switch (range) {
    case 'first15':
      return katakanaData.slice(0, 15) // ア → ソ
    case 'mid15':
      return katakanaData.slice(15, 30) // タ → ホ
    case 'last16':
      return katakanaData.slice(30, 46) // マ → ン
    case 'all':
    default:
      return katakanaData
  }
}

function buildOptions(item: KatakanaCharacter, mode: 'char-to-romaji' | 'romaji-to-char', pool: KatakanaCharacter[]): string[] {
  const correct = mode === 'char-to-romaji' ? item.romaji : item.character
  const filteredPool = pool.filter(k => k.character !== item.character)
  const shuffledPool = shuffle(filteredPool)
  const distractors = shuffledPool.slice(0, 3).map(k =>
    mode === 'char-to-romaji' ? k.romaji : k.character
  )
  return shuffle([correct, ...distractors])
}

function resolveMode(selected: QuizMode): 'char-to-romaji' | 'romaji-to-char' {
  if (selected === 'random') return Math.random() < 0.5 ? 'char-to-romaji' : 'romaji-to-char'
  return selected
}

function makeQuestion(
  item: KatakanaCharacter,
  selected: QuizMode,
  pool: KatakanaCharacter[],
  isReview = false,
): KataQuestion {
  const mode = resolveMode(selected)
  return {
    item,
    mode,
    options: buildOptions(item, mode, pool),
    correct: mode === 'char-to-romaji' ? item.romaji : item.character,
    isReview,
  }
}

function buildQuiz(selectedMode: QuizMode, selectedRange: WordRange, selectedOrder: WordOrder): KataQuestion[] {
  const baseList = getRangeData(selectedRange)
  const list = selectedOrder === 'shuffle' ? shuffle(baseList) : [...baseList]
  return list.map(item => makeQuestion(item, selectedMode, baseList))
}

// ── State ────────────────────────────────────────────────────────────────────
const quizMode    = ref<QuizMode>('random')
const wordRange   = ref<WordRange>('all')
const wordOrder   = ref<WordOrder>('ordered')

const quizOrder   = ref<KataQuestion[]>([])
const currentIdx  = ref(0)
const answerState  = ref<AnswerState>('idle')
const selected     = ref<string | null>(null)
const score        = ref({ correct: 0, incorrect: 0 })
const isFinished   = ref(false)
const cardKey      = ref(0)

const rangeOptions: { value: WordRange; label: string; sub: string }[] = [
  { value: 'all',     label: 'Tất cả 46', sub: 'ア - ン' },
  { value: 'first15', label: '15 từ đầu', sub: 'ア - ソ' },
  { value: 'mid15',   label: '15 từ giữa', sub: 'タ - ホ' },
  { value: 'last16',  label: '16 từ cuối', sub: 'マ - ン' },
]

const orderOptions: { value: WordOrder; label: string; icon: string }[] = [
  { value: 'ordered', label: 'Theo thứ tự', icon: '📋' },
  { value: 'shuffle', label: 'Xáo trộn',   icon: '🔀' },
]

const modeOptions: { value: QuizMode; label: string; icon: string }[] = [
  { value: 'char-to-romaji', label: 'ア → Romaji', icon: '🇯🇵' },
  { value: 'romaji-to-char', label: 'Romaji → ア', icon: '🇻🇳' },
  { value: 'random',         label: 'Lộn xộn',     icon: '🎲' },
]

// ── Derived ──────────────────────────────────────────────────────────────────
const total    = computed(() => quizOrder.value.filter(item => !item.isReview).length)
const question = computed(() => quizOrder.value[currentIdx.value] ?? null)
const progress = computed(() => {
  if (isFinished.value) return total.value
  return score.value.correct + score.value.incorrect
})
const isLastQuestion = computed(() => currentIdx.value >= quizOrder.value.length - 1)
const accuracy = computed(() => {
  const done = score.value.correct + score.value.incorrect
  return done === 0 ? 0 : Math.round((score.value.correct / done) * 100)
})

// ── Actions ──────────────────────────────────────────────────────────────────
function startQuiz() {
  quizOrder.value   = buildQuiz(quizMode.value, wordRange.value, wordOrder.value)
  currentIdx.value  = 0
  answerState.value = 'idle'
  selected.value    = null
  score.value       = { correct: 0, incorrect: 0 }
  isFinished.value  = false
  cardKey.value     = 0
}

function onModeChange(mode: QuizMode) {
  if (quizMode.value === mode) return
  quizMode.value = mode
  startQuiz()
}

function onRangeChange(range: WordRange) {
  if (wordRange.value === range) return
  wordRange.value = range
  startQuiz()
}

function onOrderChange(order: WordOrder) {
  if (wordOrder.value === order) return
  wordOrder.value = order
  startQuiz()
}

function playAudio() {
  if (!question.value) return
  speakJapaneseWord(question.value.item.character, question.value.item.romaji)
}

function scheduleReview(questionToReview: KataQuestion) {
  const pool = getRangeData(wordRange.value)
  const reviewQuestion = makeQuestion(questionToReview.item, questionToReview.mode, pool, true)
  // Chèn sau 2 lượt khác nếu còn đủ câu; ở cuối hàng đợi thì chèn ngay sau câu hiện tại.
  const reviewIndex = Math.min(currentIdx.value + 3, quizOrder.value.length)
  quizOrder.value.splice(reviewIndex, 0, reviewQuestion)
}

function onSelect(option: string) {
  if (answerState.value !== 'idle' || !question.value) return
  const currentQuestion = question.value
  selected.value = option

  if (option === currentQuestion.correct) {
    answerState.value = 'correct'
    if (!currentQuestion.isReview) {
      score.value = { ...score.value, correct: score.value.correct + 1 }
    }
    // Phát âm chuẩn khi trả lời đúng
    playAudio()
  } else {
    answerState.value = 'incorrect'
    if (!currentQuestion.isReview) {
      score.value = { ...score.value, incorrect: score.value.incorrect + 1 }
    }
    scheduleReview(currentQuestion)
  }
}

function onNext() {
  if (isLastQuestion.value) {
    isFinished.value = true
    return
  }
  currentIdx.value++
  answerState.value = 'idle'
  selected.value    = null
  cardKey.value++
}


function handleKeydown(e: KeyboardEvent) {
  if (e.target instanceof HTMLInputElement || e.target instanceof HTMLTextAreaElement) return

  // Phím 1–4: chọn đáp án
  if (['1', '2', '3', '4'].includes(e.key) && answerState.value === 'idle' && question.value) {
    const idx = parseInt(e.key) - 1
    const opt = question.value.options[idx]
    if (opt !== undefined) onSelect(opt)
    return
  }

  // Enter hoặc Space: qua câu tiếp theo
  if ((e.key === 'Enter' || e.key === ' ') && answerState.value !== 'idle') {
    e.preventDefault()
    onNext()
  }
}

onMounted(() => window.addEventListener('keydown', handleKeydown))
onUnmounted(() => window.removeEventListener('keydown', handleKeydown))

startQuiz()
</script>

<template>
  <div class="mobile-quiz-wrapper">
    <!-- Top Navigation Bar -->
    <div class="top-nav">
      <a href="#/" class="back-pill ios-pressable">
        <svg xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke-width="2.5" stroke="currentColor" class="w-4 h-4">
          <path stroke-linecap="round" stroke-linejoin="round" d="M15.75 19.5 8.25 12l7.5-7.5" />
        </svg>
        <span>Menu</span>
      </a>

      <div class="quiz-title-badge">
        <span class="badge-dot"></span>
        <span>Katakana Quiz</span>
      </div>

      <div class="counter-badge">
        {{ progress }}/{{ total }}
      </div>
    </div>

    <!-- Filter & Control Section -->
    <div class="controls-card ios-card">
      <!-- 1. Phạm vi chọn từ -->
      <div class="control-group">
        <div class="control-label-row">
          <span class="control-label-icon">🎯</span>
          <span class="control-label-text">Chọn phần học:</span>
        </div>
        <div class="range-grid">
          <button
            v-for="opt in rangeOptions"
            :key="opt.value"
            type="button"
            class="range-btn ios-pressable"
            :class="{ 'range-active': wordRange === opt.value }"
            @click="onRangeChange(opt.value)"
          >
            <span class="range-btn-title">{{ opt.label }}</span>
            <span class="range-btn-sub">{{ opt.sub }}</span>
          </button>
        </div>
      </div>

      <div class="control-divider"></div>

      <!-- 2. Sắp xếp từ & Kiểu trắc nghiệm -->
      <div class="control-sub-row">
        <!-- Thứ tự từ -->
        <div class="mini-control-group">
          <div class="mini-label">
            <span>Sắp xếp từ:</span>
          </div>
          <div class="order-segmented">
            <button
              v-for="ord in orderOptions"
              :key="ord.value"
              type="button"
              class="order-tab"
              :class="{ 'order-tab-active': wordOrder === ord.value }"
              @click="onOrderChange(ord.value)"
            >
              <span>{{ ord.icon }}</span>
              <span>{{ ord.label }}</span>
            </button>
          </div>
        </div>

        <!-- Chế độ trắc nghiệm -->
        <div class="mini-control-group">
          <div class="mini-label">
            <span>Kiểu câu hỏi:</span>
          </div>
          <div class="mode-segmented">
            <button
              v-for="opt in modeOptions"
              :key="opt.value"
              type="button"
              class="mode-tab"
              :class="{ 'mode-tab-active': quizMode === opt.value }"
              @click="onModeChange(opt.value)"
            >
              <span class="mode-icon">{{ opt.icon }}</span>
              <span class="mode-text">{{ opt.label }}</span>
            </button>
          </div>
        </div>
      </div>
    </div>

    <!-- Progress Line -->
    <div class="progress-bar-container" v-if="total > 0 && !isFinished">
      <div class="progress-line-track">
        <div class="progress-line-fill" :style="{ width: `${(progress / total) * 100}%` }"></div>
      </div>
    </div>

    <!-- RESULT SCREEN -->
    <div v-if="isFinished" class="result-box ios-card">
      <div class="result-sparkle">
        {{ accuracy >= 80 ? '🏆' : accuracy >= 50 ? '🌟' : '💪' }}
      </div>
      <h2 class="result-headline">
        {{ accuracy >= 80 ? 'Xuất Sắc!' : accuracy >= 50 ? 'Làm Tốt Lắm!' : 'Cố Gắng Thêm Nhé!' }}
      </h2>
      <p class="result-desc">
        Bạn đã hoàn thành {{ total }} ký tự Katakana
        <span v-if="wordRange === 'first15'">(15 từ đầu)</span>
        <span v-else-if="wordRange === 'mid15'">(15 từ giữa)</span>
        <span v-else-if="wordRange === 'last16'">(16 từ cuối)</span>
      </p>

      <div class="stats-row">
        <div class="stat-pill correct-pill">
          <span class="stat-digit">{{ score.correct }}</span>
          <span class="stat-tag">Đúng</span>
        </div>
        <div class="stat-pill incorrect-pill">
          <span class="stat-digit">{{ score.incorrect }}</span>
          <span class="stat-tag">Sai</span>
        </div>
        <div class="stat-pill acc-pill">
          <span class="stat-digit">{{ accuracy }}%</span>
          <span class="stat-tag">Chính xác</span>
        </div>
      </div>

      <div class="result-actions">
        <button class="primary-btn restart-action ios-pressable" @click="startQuiz">
          <svg xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke-width="2.5" stroke="currentColor" class="w-5 h-5">
            <path stroke-linecap="round" stroke-linejoin="round" d="M16.023 9.348h4.992v-.001M2.985 19.644v-4.992m0 0h4.992m-4.993 0 3.181 3.183a8.25 8.25 0 0 0 13.803-3.7M4.031 9.865a8.25 8.25 0 0 1 13.803-3.7l3.181 3.182m0-4.991v4.99" />
          </svg>
          Luyện Tập Lại
        </button>
      </div>
    </div>

    <!-- QUESTION CARD -->
    <Transition v-else-if="question" name="card" mode="out-in">
      <div :key="cardKey" class="quiz-card ios-card">
        <!-- Question Header Badge & Audio button -->
        <div class="card-top-info">
          <span class="direction-badge" :class="question.mode === 'char-to-romaji' ? 'dir-char' : 'dir-roma'">
            {{ question.mode === 'char-to-romaji' ? '🇯🇵 Nhìn chữ → Chọn Romaji' : '🇻🇳 Nhìn Romaji → Chọn chữ' }}
          </span>
          <button
            type="button"
            class="mini-audio-btn ios-pressable"
            @click="playAudio"
            title="Nghe phát âm"
          >
            <svg xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke-width="2.2" stroke="currentColor" class="w-4 h-4">
              <path stroke-linecap="round" stroke-linejoin="round" d="M19.114 5.636a9 9 0 0 1 0 12.728M16.463 8.288a5.25 5.25 0 0 1 0 7.424M6.75 8.25l4.72-4.72a.75.75 0 0 1 1.28.53v15.88a.75.75 0 0 1-1.28.53l-4.72-4.72H4.51c-.88 0-1.704-.507-1.938-1.354A9.009 9.009 0 0 1 2.25 12c0-.83.112-1.633.322-2.396C2.806 8.756 3.63 8.25 4.51 8.25H6.75Z" />
            </svg>
          </button>
        </div>

        <!-- Big Prompt Display -->
        <div class="prompt-box" @click="playAudio" title="Chạm để nghe âm thanh">
          <span
            class="main-prompt"
            :class="question.mode === 'char-to-romaji' ? 'prompt-kana' : 'prompt-latin'"
          >
            {{ question.mode === 'char-to-romaji' ? question.item.character : question.item.romaji }}
          </span>
          <span class="prompt-speaker-hint">🔊 Chạm để nghe</span>
        </div>

        <p class="select-hint">Chọn 1 trong 4 đáp án:</p>

        <!-- 4 Options Grid (2x2 touch optimized) -->
        <div class="options-container">
          <button
            v-for="(opt, idx) in question.options"
            :key="opt"
            type="button"
            class="choice-btn ios-pressable"
            :class="{
              'choice-correct':  answerState !== 'idle' && opt === question.correct,
              'choice-wrong':    answerState !== 'idle' && opt === selected && opt !== question.correct,
              'choice-dimmed':   answerState !== 'idle' && opt !== selected && opt !== question.correct,
              'choice-kana':     question.mode === 'romaji-to-char',
            }"
            :disabled="answerState !== 'idle'"
            @click="onSelect(opt)"
          >
            <span class="choice-key-badge">{{ idx + 1 }}</span>
            {{ opt }}
          </button>
        </div>

        <!-- Bottom Feedback Floating Bar -->
        <Transition name="feedback">
          <div v-if="answerState !== 'idle'" class="feedback-panel">
            <div class="feedback-status">
              <span v-if="answerState === 'correct'" class="status-icon-correct">✓</span>
              <span v-else class="status-icon-wrong">✕</span>
              <div class="status-text">
                <p v-if="answerState === 'correct'" class="status-title correct-title">Chính xác!</p>
                <p v-else class="status-title wrong-title">
                  Đáp án đúng: <span class="highlight-ans">{{ question.correct }}</span>
                </p>
              </div>
            </div>

            <button type="button" class="primary-btn next-action ios-pressable" @click="onNext">
              <span>{{ isLastQuestion ? 'Xem kết quả' : 'Tiếp theo' }}</span>
              <svg xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke-width="2.5" stroke="currentColor" class="w-4 h-4">
                <path stroke-linecap="round" stroke-linejoin="round" d="M13.5 4.5 21 12m0 0-7.5 7.5M21 12H3" />
              </svg>
            </button>
          </div>
        </Transition>
      </div>
    </Transition>
  </div>
</template>

<style scoped>
.mobile-quiz-wrapper {
  min-height: 100dvh;
  max-width: 480px;
  margin: 0 auto;
  padding: calc(var(--sat) + 0.75rem) 0.85rem calc(var(--sab) + 1.25rem);
  display: flex;
  flex-direction: column;
  gap: 0.75rem;
  box-sizing: border-box;
}

/* ── Top Bar ── */
.top-nav {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 0.5rem;
}

.back-pill {
  display: inline-flex;
  align-items: center;
  gap: 0.3rem;
  font-size: 0.82rem;
  font-weight: 700;
  color: var(--color-ink);
  text-decoration: none;
  padding: 0.45rem 0.85rem;
  border-radius: 999px;
  background: rgba(255, 255, 255, 0.85);
  backdrop-filter: blur(12px);
  -webkit-backdrop-filter: blur(12px);
  border: 1px solid rgba(255, 255, 255, 0.9);
  box-shadow: 0 2px 8px rgba(15, 23, 42, 0.04);
}

/* Katakana quiz — màu cyan/teal để phân biệt với Hiragana (tím) */
.quiz-title-badge {
  display: inline-flex;
  align-items: center;
  gap: 0.35rem;
  font-size: 0.78rem;
  font-weight: 800;
  color: #0e7490;
  background: rgba(236, 254, 255, 0.85);
  padding: 0.35rem 0.75rem;
  border-radius: 999px;
  border: 1px solid rgba(165, 243, 252, 0.8);
}

.badge-dot {
  width: 6px;
  height: 6px;
  border-radius: 50%;
  background: #06b6d4;
}

.counter-badge {
  font-size: 0.8rem;
  font-weight: 800;
  color: var(--color-muted);
  background: rgba(255, 255, 255, 0.85);
  padding: 0.4rem 0.75rem;
  border-radius: 999px;
  border: 1px solid rgba(255, 255, 255, 0.9);
}

/* ── Filter Controls Card ── */
.controls-card {
  padding: 0.75rem 0.85rem;
  border-radius: 20px;
  display: flex;
  flex-direction: column;
  gap: 0.6rem;
  background: rgba(255, 255, 255, 0.88);
}

.control-group {
  display: flex;
  flex-direction: column;
  gap: 0.4rem;
}

.control-label-row {
  display: flex;
  align-items: center;
  gap: 0.3rem;
}

.control-label-icon {
  font-size: 0.8rem;
}

.control-label-text {
  font-size: 0.72rem;
  font-weight: 800;
  color: #475569;
  text-transform: uppercase;
  letter-spacing: 0.04em;
}

/* Range grid (4 buttons) */
.range-grid {
  display: grid;
  grid-template-columns: repeat(4, 1fr);
  gap: 4px;
}

.range-btn {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  padding: 0.45rem 0.2rem;
  border: 1.5px solid #e2e8f0;
  background: #f8fafc;
  border-radius: 12px;
  cursor: pointer;
  font-family: inherit;
  transition: all 0.18s cubic-bezier(0.4, 0, 0.2, 1);
  user-select: none;
  -webkit-user-select: none;
}

.range-btn-title {
  font-size: 0.68rem;
  font-weight: 800;
  color: #334155;
  white-space: nowrap;
}

.range-btn-sub {
  font-size: 0.58rem;
  font-weight: 700;
  color: #94a3b8;
  margin-top: 1px;
}

/* Katakana range-active — teal thay vì tím */
.range-active {
  background: linear-gradient(145deg, #0e7490 0%, #06b6d4 100%);
  border-color: #0e7490;
  box-shadow: 0 3px 8px rgba(14, 116, 144, 0.28);
}

.range-active .range-btn-title {
  color: #ffffff;
}

.range-active .range-btn-sub {
  color: #cffafe;
}

.control-divider {
  height: 1px;
  background: rgba(226, 232, 240, 0.7);
  margin: 0.1rem 0;
}

/* Sub-row for Order & Mode */
.control-sub-row {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}

.mini-control-group {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 0.5rem;
}

.mini-label {
  font-size: 0.7rem;
  font-weight: 700;
  color: #64748b;
  white-space: nowrap;
}

/* Order Segmented */
.order-segmented {
  display: flex;
  gap: 4px;
  background: rgba(241, 245, 249, 0.95);
  padding: 3px;
  border-radius: 10px;
  border: 1px solid #e2e8f0;
}

.order-tab {
  display: flex;
  align-items: center;
  gap: 0.25rem;
  padding: 0.3rem 0.55rem;
  border: none;
  background: transparent;
  border-radius: 8px;
  cursor: pointer;
  font-family: inherit;
  font-size: 0.68rem;
  font-weight: 700;
  color: #64748b;
  transition: all 0.15s ease;
  user-select: none;
}

.order-tab-active {
  background: #ffffff;
  color: #0e7490;
  font-weight: 800;
  box-shadow: 0 2px 6px rgba(15, 23, 42, 0.08);
}

/* Mode Segmented */
.mode-segmented {
  display: flex;
  gap: 4px;
  background: rgba(241, 245, 249, 0.95);
  padding: 3px;
  border-radius: 10px;
  border: 1px solid #e2e8f0;
}

.mode-tab {
  display: flex;
  align-items: center;
  gap: 0.2rem;
  padding: 0.3rem 0.45rem;
  border: none;
  background: transparent;
  border-radius: 8px;
  cursor: pointer;
  font-family: inherit;
  font-size: 0.66rem;
  font-weight: 700;
  color: #64748b;
  transition: all 0.15s ease;
  user-select: none;
  white-space: nowrap;
}

.mode-icon {
  font-size: 0.72rem;
}

.mode-tab-active {
  background: #ffffff;
  color: #0e7490;
  font-weight: 800;
  box-shadow: 0 2px 6px rgba(15, 23, 42, 0.08);
}

/* ── Progress Line ── */
.progress-bar-container {
  padding: 0 2px;
}

.progress-line-track {
  height: 6px;
  border-radius: 999px;
  background: rgba(226, 232, 240, 0.8);
  overflow: hidden;
}

.progress-line-fill {
  height: 100%;
  border-radius: 999px;
  /* Katakana: teal → sky gradient */
  background: linear-gradient(90deg, #0e7490, #38bdf8);
  transition: width 0.35s cubic-bezier(0.4, 0, 0.2, 1);
}

/* ── Question Card ── */
.quiz-card {
  border-radius: 28px;
  padding: 1.25rem 1.15rem 1.15rem;
  display: flex;
  flex-direction: column;
  gap: 0.85rem;
  position: relative;
  overflow: hidden;
}

.card-top-info {
  display: flex;
  align-items: center;
  justify-content: space-between;
}

.direction-badge {
  font-size: 0.7rem;
  font-weight: 800;
  padding: 0.3rem 0.75rem;
  border-radius: 999px;
  letter-spacing: 0.02em;
}

.dir-char {
  background: #ecfeff;
  color: #0e7490;
  border: 1px solid #a5f3fc;
}

.dir-roma {
  background: #eff6ff;
  color: #1d4ed8;
  border: 1px solid #dbeafe;
}

.mini-audio-btn {
  width: 32px;
  height: 32px;
  border-radius: 50%;
  border: 1px solid #e2e8f0;
  background: #f8fafc;
  color: #0e7490;
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
}

/* ── Big Prompt Box ── */
.prompt-box {
  background: linear-gradient(180deg, #f8fafc 0%, #f1f5f9 100%);
  border-radius: 20px;
  border: 1px solid rgba(226, 232, 240, 0.8);
  padding: 1.15rem 0.5rem 0.65rem;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  min-height: 120px;
  box-shadow: inset 0 2px 4px rgba(0, 0, 0, 0.02);
  cursor: pointer;
}

.main-prompt {
  line-height: 1;
  font-weight: 900;
}

.prompt-speaker-hint {
  font-size: 0.65rem;
  font-weight: 700;
  color: #94a3b8;
  margin-top: 0.4rem;
}

/* Katakana prompt — teal gradient thay vì tím */
.prompt-kana {
  font-size: clamp(4.25rem, 16vw, 5.25rem);
  font-family: 'M PLUS Rounded 1c', sans-serif;
  background: linear-gradient(145deg, #0e7490 0%, #06b6d4 100%);
  -webkit-background-clip: text;
  background-clip: text;
  -webkit-text-fill-color: transparent;
}

.prompt-latin {
  font-size: clamp(2.25rem, 9vw, 3rem);
  letter-spacing: 0.05em;
  color: #1e40af;
}

.select-hint {
  font-size: 0.75rem;
  font-weight: 700;
  color: var(--color-muted);
  text-align: center;
  margin: -0.25rem 0 0;
}

/* ── Options 2x2 ── */
.options-container {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 0.6rem;
}

.choice-btn {
  min-height: 52px;
  padding: 0.65rem 0.5rem;
  border-radius: 16px;
  border: 2px solid #e2e8f0;
  background: #ffffff;
  font-family: inherit;
  font-size: 1.15rem;
  font-weight: 800;
  color: var(--color-ink);
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  box-shadow: 0 2px 6px rgba(15, 23, 42, 0.02);
  position: relative;
}

.choice-key-badge {
  position: absolute;
  top: 5px;
  left: 7px;
  width: 18px;
  height: 18px;
  border-radius: 6px;
  background: #e0e7ff;
  color: #4338ca;
  font-size: 0.6rem;
  font-weight: 900;
  display: flex;
  align-items: center;
  justify-content: center;
  line-height: 1;
  letter-spacing: 0;
  transition: background 0.15s ease, color 0.15s ease;
}

.choice-correct .choice-key-badge {
  background: #bbf7d0;
  color: #047857;
}

.choice-wrong .choice-key-badge {
  background: #fecdd3;
  color: #be123c;
}

.choice-kana {
  font-family: 'M PLUS Rounded 1c', sans-serif;
  font-size: 1.55rem;
}

.choice-correct {
  background: #ecfdf5 !important;
  border-color: #10b981 !important;
  color: #047857 !important;
}

.choice-wrong {
  background: #fef2f2 !important;
  border-color: #ef4444 !important;
  color: #b91c1c !important;
}

.choice-dimmed {
  opacity: 0.4;
}

/* ── Feedback Panel (Fixed Bottom Sheet) ── */
.feedback-panel {
  position: fixed;
  bottom: 0;
  left: 50%;
  transform: translateX(-50%);
  width: 100%;
  max-width: 480px;
  padding: 1.25rem 1.25rem calc(var(--sab, env(safe-area-inset-bottom)) + 1.25rem);
  background: rgba(255, 255, 255, 0.98);
  backdrop-filter: blur(20px);
  -webkit-backdrop-filter: blur(20px);
  border-top: 1px solid rgba(226, 232, 240, 0.8);
  border-top-left-radius: 28px;
  border-top-right-radius: 28px;
  box-shadow: 0 -10px 40px rgba(15, 23, 42, 0.1);
  display: flex;
  flex-direction: column;
  gap: 1rem;
  z-index: 100;
}

.feedback-status {
  display: flex;
  align-items: center;
  gap: 0.6rem;
}

.status-icon-correct {
  width: 28px;
  height: 28px;
  border-radius: 50%;
  background: #dcfce7;
  color: #15803d;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 0.9rem;
  font-weight: 900;
  flex-shrink: 0;
}

.status-icon-wrong {
  width: 28px;
  height: 28px;
  border-radius: 50%;
  background: #fee2e2;
  color: #dc2626;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 0.9rem;
  font-weight: 900;
  flex-shrink: 0;
}

.status-text { display: flex; flex-direction: column; }

.status-title {
  font-size: 0.85rem;
  font-weight: 800;
  margin: 0;
  line-height: 1.2;
}

.correct-title { color: #15803d; }
.wrong-title   { color: #dc2626; }

.status-subtitle {
  font-size: 0.72rem;
  font-weight: 600;
  color: #64748b;
  margin: 0.1rem 0 0;
  line-height: 1.2;
}

/* highlight-ans dùng màu teal của Katakana */

.highlight-ans {
  font-family: 'M PLUS Rounded 1c', sans-serif;
  font-size: 1rem;
  color: #0e7490;
}

/* ── Buttons ── */
.primary-btn {
  width: 100%;
  min-height: 48px;
  border-radius: 16px;
  border: none;
  background: linear-gradient(135deg, #0e7490 0%, #06b6d4 100%);
  color: #ffffff;
  font-family: inherit;
  font-size: 0.92rem;
  font-weight: 800;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 0.4rem;
  box-shadow: 0 6px 20px rgba(14, 116, 144, 0.35);
}

/* ── Result Screen ── */
.result-box {
  border-radius: 28px;
  padding: 2rem 1.5rem;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 1rem;
  text-align: center;
}

.result-sparkle {
  font-size: 3.5rem;
  line-height: 1;
}

.result-headline {
  font-size: 1.6rem;
  font-weight: 900;
  color: var(--color-ink);
  margin: 0;
  letter-spacing: -0.02em;
}

.result-desc {
  font-size: 0.875rem;
  color: var(--color-muted);
  margin: 0;
  font-weight: 500;
}

.stats-row {
  display: flex;
  gap: 0.6rem;
}

.stat-pill {
  flex: 1;
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 0.75rem 0.5rem;
  border-radius: 16px;
  gap: 0.15rem;
}

.correct-pill  { background: #ecfdf5; border: 1.5px solid #a7f3d0; }
.incorrect-pill{ background: #fef2f2; border: 1.5px solid #fecaca; }
.acc-pill      { background: #ecfeff; border: 1.5px solid #a5f3fc; }

.stat-digit {
  font-size: 1.45rem;
  font-weight: 900;
  color: var(--color-ink);
  line-height: 1;
}

.stat-tag {
  font-size: 0.65rem;
  font-weight: 700;
  color: var(--color-muted);
  text-transform: uppercase;
  letter-spacing: 0.04em;
}

.result-actions { width: 100%; }

.restart-action {
  width: 100%;
  justify-content: center;
  padding: 0.85rem;
  font-size: 0.95rem;
  border-radius: 18px;
}

/* ── Card Transition ── */
.card-enter-active,
.card-leave-active {
  transition: opacity 0.2s ease, transform 0.2s ease;
}
.card-enter-from {
  opacity: 0;
  transform: translateX(24px);
}
.card-leave-to {
  opacity: 0;
  transform: translateX(-24px);
}

/* ── Feedback Transition ── */
.feedback-enter-active,
.feedback-leave-active {
  transition: opacity 0.3s ease, transform 0.3s cubic-bezier(0.16, 1, 0.3, 1);
}
.feedback-enter-from,
.feedback-leave-to {
  opacity: 0;
  transform: translate(-50%, 100%);
}
.feedback-enter-to {
  opacity: 1;
  transform: translate(-50%, 0);
}

/* ── iOS Pressable ── */
.ios-pressable {
  transition: transform 0.12s ease, opacity 0.12s ease;
}
.ios-pressable:active {
  transform: scale(0.95);
  opacity: 0.85;
}

/* ── iOS Card ── */
.ios-card {
  background: rgba(255, 255, 255, 0.88);
  backdrop-filter: blur(20px);
  -webkit-backdrop-filter: blur(20px);
  border: 1px solid rgba(255, 255, 255, 0.9);
  box-shadow: 0 8px 32px rgba(15, 23, 42, 0.06), 0 2px 8px rgba(15, 23, 42, 0.03);
}
</style>
