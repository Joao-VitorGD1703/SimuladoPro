<script setup>
import { ref, watch, onMounted, onUnmounted } from 'vue'

const questions = ref([])
const userAnswers = ref({})
const hasSubmitted = ref(false)
const score = ref(0)
const errorMsg = ref('')
const currentQuestionIndex = ref(0)

onMounted(() => {
  const cachedQuestions = localStorage.getItem('simulado_questions')
  const cachedAnswers = localStorage.getItem('simulado_user_answers')
  const cachedIndex = localStorage.getItem('simulado_current_index')
  const cachedSubmitted = localStorage.getItem('simulado_has_submitted')
  
  if (cachedQuestions) {
    questions.value = JSON.parse(cachedQuestions)
  }
  if (cachedAnswers) {
    userAnswers.value = JSON.parse(cachedAnswers)
  }
  if (cachedIndex) {
    currentQuestionIndex.value = parseInt(cachedIndex, 10)
  }
  if (cachedSubmitted === 'true') {
    hasSubmitted.value = true
    let correct = 0
    questions.value.forEach((q, index) => {
      if (userAnswers.value[index] === q.answer) {
        correct++
      }
    })
    score.value = correct
  }

  window.addEventListener('keydown', handleKeydown)
})

onUnmounted(() => {
  window.removeEventListener('keydown', handleKeydown)
})

const handleKeydown = (e) => {
  if (questions.value.length > 0 && !hasSubmitted.value) {
    if (e.key === 'ArrowRight') {
      nextQuestion()
    } else if (e.key === 'ArrowLeft') {
      prevQuestion()
    }
  }
}

// Cache watchers
watch(questions, (newVal) => {
  if (newVal.length > 0) {
    localStorage.setItem('simulado_questions', JSON.stringify(newVal))
  } else {
    localStorage.removeItem('simulado_questions')
  }
}, { deep: true })

watch(userAnswers, (newVal) => {
  localStorage.setItem('simulado_user_answers', JSON.stringify(newVal))
}, { deep: true })

watch(currentQuestionIndex, (newVal) => {
  localStorage.setItem('simulado_current_index', newVal.toString())
})

const inputMode = ref('upload')
const pastedJson = ref('')
const isDragging = ref(false)

const parseAndLoadJson = (text) => {
  try {
    let cleanText = text.trim();
    if (cleanText.charCodeAt(0) === 0xFEFF) {
      cleanText = cleanText.slice(1);
    }
    const data = JSON.parse(cleanText)
    
    if (Array.isArray(data)) {
      const isValid = data.every(q => 
        q && 
        typeof q.question === 'string' && 
        Array.isArray(q.options) && 
        q.options.length > 0 &&
        typeof q.answer !== 'undefined'
      );

      if (!isValid) {
        errorMsg.value = 'Formatação inválida. O arquivo deve conter uma lista onde cada item tem "question", "options" (como lista) e "answer".'
        return
      }

      questions.value = data
      userAnswers.value = {}
      hasSubmitted.value = false
      errorMsg.value = ''
      currentQuestionIndex.value = 0
    } else {
      errorMsg.value = 'O arquivo JSON não é válido. Ele deve começar com um colchete [ (Array).'
    }
  } catch (err) {
    console.error(err)
    errorMsg.value = `Erro de leitura JSON: ${err.message}`
  }
}

const handleFileUpload = (event) => {
  let file;
  if (event.target && event.target.files) {
    file = event.target.files[0]
  } else if (event.dataTransfer && event.dataTransfer.files) {
    file = event.dataTransfer.files[0]
  }

  if (!file) return
  
  const reader = new FileReader()
  reader.onload = (e) => {
    parseAndLoadJson(e.target.result)
    if (event.target && event.target.type === 'file') event.target.value = ''
  }
  reader.readAsText(file)
}

const handleDrop = (event) => {
  isDragging.value = false
  handleFileUpload(event)
}

const loadFromPaste = () => {
  if (!pastedJson.value.trim()) {
    errorMsg.value = 'Cole o JSON antes de carregar.'
    return
  }
  parseAndLoadJson(pastedJson.value)
}

const onAnswerSelect = () => {
  if (!hasSubmitted.value && currentQuestionIndex.value < questions.value.length - 1) {
    setTimeout(() => {
      nextQuestion()
    }, 400)
  }
}

const prevQuestion = () => {
  if (currentQuestionIndex.value > 0) {
    currentQuestionIndex.value--
  }
}

const nextQuestion = () => {
  if (currentQuestionIndex.value < questions.value.length - 1) {
    currentQuestionIndex.value++
  }
}

const submitQuiz = () => {
  let correct = 0
  questions.value.forEach((q, index) => {
    if (userAnswers.value[index] === q.answer) {
      correct++
    }
  })
  score.value = correct
  hasSubmitted.value = true
  localStorage.setItem('simulado_has_submitted', 'true')
  
  // Smooth scroll to bottom to see results
  setTimeout(() => {
    window.scrollTo({ top: document.body.scrollHeight, behavior: 'smooth' })
  }, 100)
}

const resetQuiz = () => {
  questions.value = []
  userAnswers.value = {}
  hasSubmitted.value = false
  score.value = 0
  errorMsg.value = ''
  currentQuestionIndex.value = 0
  localStorage.removeItem('simulado_questions')
  localStorage.removeItem('simulado_user_answers')
  localStorage.removeItem('simulado_current_index')
  localStorage.removeItem('simulado_has_submitted')
  window.scrollTo({ top: 0, behavior: 'smooth' })
}
</script>

<template>
  <div class="app-wrapper">
    <cv-content class="main-content">
      <transition name="fade" mode="out-in">
        
        <!-- START SCREEN -->
        <div v-if="questions.length === 0" class="onboarding-screen">
          <div class="header-text">
            <h1 class="title">Simulado<span class="highlight">Pro</span></h1>
            <p class="subtitle">O seu ambiente de testes rápido e eficiente.</p>
          </div>
          
          <cv-inline-notification
            v-if="errorMsg"
            kind="error"
            :title="errorMsg"
            @close="errorMsg = ''"
            class="error-notification"
          />

          <div class="input-mode-toggle">
            <button :class="['toggle-btn', { active: inputMode === 'upload' }]" @click="inputMode = 'upload'">Fazer Upload</button>
            <button :class="['toggle-btn', { active: inputMode === 'paste' }]" @click="inputMode = 'paste'">Colar JSON</button>
          </div>

          <!-- UPLOAD MODE -->
          <div 
            v-if="inputMode === 'upload'"
            class="upload-card"
            :class="{ 'is-dragging': isDragging }"
            @dragover.prevent="isDragging = true"
            @dragleave.prevent="isDragging = false"
            @drop.prevent="handleDrop"
          >
            <div class="upload-icon">
              <svg focusable="false" preserveAspectRatio="xMidYMid meet" xmlns="http://www.w3.org/2000/svg" fill="currentColor" width="48" height="48" viewBox="0 0 32 32" aria-hidden="true"><path d="M26,22v4H6V22H4v4a2,2,0,0,0,2,2H26a2,2,0,0,0,2-2V22ZM16,4,6.46,13.54l1.42,1.41L15,7.83V20h2V7.83l7.12,7.12,1.42-1.41Z"></path></svg>
            </div>
            <h3>Importe suas questões</h3>
            <p class="upload-desc">Arraste e solte seu arquivo <strong>.json</strong> aqui ou clique para selecionar.</p>
            
            <input type="file" accept=".json" @change="handleFileUpload" id="file-upload" class="file-input-hidden" />
            <label for="file-upload" class="upload-btn">Selecionar Arquivo</label>
          </div>

          <!-- PASTE MODE -->
          <div v-else class="paste-card">
            <h3>Cole seu JSON</h3>
            <textarea 
              v-model="pastedJson" 
              class="json-textarea" 
              placeholder='Ex: [ { "question": "...", "options": ["A", "B", "C"], "answer": "B" } ]'
            ></textarea>
            <button class="upload-btn" @click="loadFromPaste">Carregar Simulado</button>
          </div>
          
          <div class="json-format-hint">
            <p>Formato de exemplo:</p>
            <pre><code>[
  {
    "question": "Qual é a capital do Brasil?",
    "options": ["Rio de Janeiro", "São Paulo", "Brasília", "Salvador"],
    "answer": "Brasília"
  },
  {
    "question": "Quanto é 2 + 2?",
    "options": ["3", "4", "5", "6"],
    "answer": "4"
  }
]</code></pre>
          </div>
        </div>

        <!-- QUIZ SCREEN -->
        <div v-else class="quiz-screen">
          <div class="quiz-header">
            <div class="quiz-header-info">
              <h2>Simulado em andamento</h2>
              <span class="badge">{{ questions.length }} questões</span>
            </div>
            <cv-button kind="danger" size="small" @click="resetQuiz" class="cancel-btn">Encerrar</cv-button>
          </div>

          <div class="questions-list">
            <template v-for="(q, index) in questions" :key="index">
              <div 
                v-if="hasSubmitted || index === currentQuestionIndex"
                class="question-card fade-in-up" 
                :class="{'is-submitted': hasSubmitted, 'is-correct': hasSubmitted && userAnswers[index] === q.answer, 'is-wrong': hasSubmitted && userAnswers[index] !== q.answer}"
                :style="{ animationDelay: hasSubmitted ? `${index * 0.05}s` : '0s' }"
              >
                <div class="question-header">
                  <div class="question-number">{{ index + 1 }}</div>
                  <h3 class="question-text">{{ q.question }}</h3>
                </div>
                
                <div class="options-container">
                  <fieldset class="bx--fieldset">
                    <div class="bx--radio-button-group">
                      <div class="bx--radio-button-wrapper" v-for="(opt, oIndex) in q.options" :key="oIndex">
                        <input type="radio" 
                          :id="`radio-${index}-${oIndex}`" 
                          class="bx--radio-button" 
                          :name="`question-${index}`" 
                          :value="opt" 
                          v-model="userAnswers[index]" 
                          :disabled="hasSubmitted" 
                          @change="onAnswerSelect" 
                        />
                        <label :for="`radio-${index}-${oIndex}`" class="bx--radio-button__label" :class="{'correct-answer': hasSubmitted && opt === q.answer, 'wrong-answer': hasSubmitted && userAnswers[index] === opt && opt !== q.answer}">
                          <span class="bx--radio-button__appearance"></span>
                          <span class="bx--radio-button__label-text">{{ opt }}</span>
                        </label>
                      </div>
                    </div>
                  </fieldset>
                </div>

                <transition name="slide-down">
                  <div v-if="hasSubmitted" class="feedback-banner">
                    <div v-if="userAnswers[index] === q.answer" class="feedback-content success">
                      <svg focusable="false" preserveAspectRatio="xMidYMid meet" xmlns="http://www.w3.org/2000/svg" fill="currentColor" width="20" height="20" viewBox="0 0 32 32"><path d="M16,2A14,14,0,1,0,30,16,14,14,0,0,0,16,2ZM14,21.59l-5-5L10.41,15,14,18.59,21.59,11,23,12.41Z"></path></svg>
                      <span>Resposta Correta!</span>
                    </div>
                    <div v-else class="feedback-content error">
                      <svg focusable="false" preserveAspectRatio="xMidYMid meet" xmlns="http://www.w3.org/2000/svg" fill="currentColor" width="20" height="20" viewBox="0 0 32 32"><path d="M16,2A14,14,0,1,0,30,16,14,14,0,0,0,16,2Zm4.59,18L19.17,21.41,16,18.24l-3.17,3.17L11.41,20l3.17-3.17L11.41,13.66,12.83,12.24,16,15.41l3.17-3.17,1.42,1.42L17.41,16.83Z"></path></svg>
                      <div class="wrong-text">
                        <span class="wrong-title">Você errou.</span> 
                        <span class="correct-is">A resposta correta é: <strong>{{ q.answer }}</strong></span>
                      </div>
                    </div>
                  </div>
                </transition>
              </div>
            </template>
          </div>

          <div class="navigation-section" v-if="!hasSubmitted">
            <button class="custom-nav-btn" :disabled="currentQuestionIndex === 0" @click="prevQuestion">
              <svg focusable="false" preserveAspectRatio="xMidYMid meet" xmlns="http://www.w3.org/2000/svg" fill="currentColor" width="16" height="16" viewBox="0 0 32 32" aria-hidden="true"><path d="M14 26L15.41 24.59 7.83 17 28 17 28 15 7.83 15 15.41 7.41 14 6 4 16 14 26z"></path></svg>
              Anterior
            </button>
            
            <div class="progress-indicator">
              {{ currentQuestionIndex + 1 }} / {{ questions.length }}
            </div>

            <button class="custom-nav-btn" :disabled="currentQuestionIndex === questions.length - 1" @click="nextQuestion">
              Próxima
              <svg focusable="false" preserveAspectRatio="xMidYMid meet" xmlns="http://www.w3.org/2000/svg" fill="currentColor" width="16" height="16" viewBox="0 0 32 32" aria-hidden="true"><path d="M18 6L16.59 7.41 24.17 15 4 15 4 17 24.17 17 16.59 24.59 18 26 28 16 18 6z"></path></svg>
            </button>
          </div>

          <div class="submit-section" v-if="!hasSubmitted && currentQuestionIndex === questions.length - 1">
            <button class="custom-submit-btn" @click="submitQuiz">
              Finalizar Simulado
              <svg focusable="false" preserveAspectRatio="xMidYMid meet" xmlns="http://www.w3.org/2000/svg" fill="currentColor" width="16" height="16" viewBox="0 0 32 32" aria-hidden="true"><path d="M18 6L16.57 7.39 24.15 15 4 15 4 17 24.15 17 16.57 24.57 18 26 28 16 18 6z"></path></svg>
            </button>
          </div>
          
          <transition name="fade">
            <div v-if="hasSubmitted" class="result-card fade-in-up">
              <div class="score-ring">
                <span class="score-percentage">{{ Math.round((score / questions.length) * 100) }}%</span>
              </div>
              <div class="score-details">
                <h2>Simulado Concluído!</h2>
                <p>Você acertou <strong>{{ score }}</strong> de <strong>{{ questions.length }}</strong> questões.</p>
                <button class="custom-outline-btn" @click="resetQuiz">
                  <svg focusable="false" preserveAspectRatio="xMidYMid meet" xmlns="http://www.w3.org/2000/svg" fill="currentColor" width="16" height="16" viewBox="0 0 32 32" aria-hidden="true"><path d="M12 29a1 1 0 01-.92-.62L6.33 17H2v-2h5a1 1 0 01.92.62L12 25.28l8.06-21.63A1 1 0 0121 3a1 1 0 01.93.68L25.72 15H30v2h-5a1 1 0 01-.95-.68L21 6.48l-8.06 21.63A1 1 0 0112 29z"></path></svg>
                  Realizar Novo Simulado
                </button>
              </div>
            </div>
          </transition>
        </div>
      </transition>
    </cv-content>
  </div>
</template>

<style scoped>
/* GENERAL STYLES */
.app-wrapper {
  min-height: 100vh;
  background-color: #f4f4f4; /* Carbon background */
  padding: 3rem 1rem;
  font-family: 'IBM Plex Sans', sans-serif;
  color: #161616;
}
.main-content {
  width: 100%;
  margin: 0 auto;
}

/* ANIMATIONS */
.fade-enter-active, .fade-leave-active {
  transition: opacity 0.4s ease;
}
.fade-enter-from, .fade-leave-to {
  opacity: 0;
}
.slide-down-enter-active, .slide-down-leave-active {
  transition: all 0.3s ease;
  overflow: hidden;
}
.slide-down-enter-from, .slide-down-leave-to {
  opacity: 0;
  max-height: 0;
}
.slide-down-enter-to, .slide-down-leave-from {
  opacity: 1;
  max-height: 100px;
}
@keyframes fadeInUp {
  from { opacity: 0; transform: translateY(20px); }
  to { opacity: 1; transform: translateY(0); }
}
.fade-in-up {
  animation: fadeInUp 0.5s ease backwards;
}

/* START SCREEN */
.onboarding-screen {
  width: 100%;
  max-width: 800px;
  margin: 0 auto;
  display: flex;
  flex-direction: column;
  align-items: center;
  text-align: center;
  padding-top: 4rem;
}
.header-text {
  margin-bottom: 3rem;
}
.title {
  font-size: 3rem;
  font-weight: 300;
  letter-spacing: -1px;
  margin-bottom: 0.5rem;
}
.highlight {
  font-weight: 600;
  color: #0f62fe; /* Carbon Blue 60 */
}
.subtitle {
  font-size: 1.125rem;
  color: #525252;
}

.input-mode-toggle {
  display: flex;
  margin-bottom: 2rem;
  background: #e0e0e0;
  border-radius: 4px;
  overflow: hidden;
  width: 100%;
  max-width: 500px;
}
.toggle-btn {
  flex: 1;
  background: transparent;
  border: none;
  padding: 1rem 2rem;
  cursor: pointer;
  font-size: 1rem;
  font-weight: 500;
  color: #525252;
  transition: all 0.2s ease;
}
.toggle-btn.active {
  background: #0f62fe;
  color: #ffffff;
}

.upload-card {
  background: #ffffff;
  border: 1px solid #e0e0e0;
  width: 100%;
  max-width: 500px;
  padding: 3rem 2rem;
  transition: all 0.3s cubic-bezier(0.2, 0.8, 0.2, 1);
  display: flex;
  flex-direction: column;
  align-items: center;
  position: relative;
  overflow: hidden;
}
.upload-card:hover {
  box-shadow: 0 4px 16px rgba(0,0,0,0.1);
  transform: translateY(-2px);
  border-color: #0f62fe;
}
.upload-icon {
  color: #0f62fe;
  margin-bottom: 1.5rem;
  transition: transform 0.3s ease;
}
.upload-card:hover .upload-icon {
  transform: translateY(-5px);
}
.upload-desc {
  color: #525252;
  margin-bottom: 2rem;
  font-size: 0.875rem;
}
.file-input-hidden {
  display: none;
}
.upload-btn {
  background-color: #0f62fe;
  color: #ffffff;
  padding: 1rem 2rem;
  cursor: pointer;
  font-weight: 500;
  transition: background-color 0.2s ease, transform 0.1s ease;
  display: inline-flex;
  align-items: center;
  gap: 0.5rem;
}
.upload-btn:hover {
  background-color: #0353e9;
}
.upload-btn:active {
  transform: scale(0.98);
}

.upload-card.is-dragging {
  border-color: #0f62fe;
  background: #f4f8ff;
  transform: scale(1.02);
}

.paste-card {
  background: #ffffff;
  border: 1px solid #e0e0e0;
  width: 100%;
  max-width: 500px;
  padding: 2rem;
  display: flex;
  flex-direction: column;
  align-items: flex-start;
  transition: all 0.3s ease;
}
.json-textarea {
  width: 100%;
  min-height: 200px;
  margin: 1.5rem 0;
  padding: 1rem;
  font-family: 'IBM Plex Mono', monospace;
  font-size: 0.875rem;
  border: 1px solid #c6c6c6;
  border-radius: 4px;
  resize: vertical;
}
.json-textarea:focus {
  outline: 2px solid #0f62fe;
  outline-offset: -2px;
}
.json-format-hint {
  margin-top: 3rem;
  text-align: left;
  background: transparent;
  width: 100%;
  max-width: 600px;
}
.json-format-hint p {
  font-size: 0.875rem;
  color: #525252;
  font-weight: 600;
  margin-bottom: 0.75rem;
}
.json-format-hint pre {
  background: #161616;
  color: #f4f4f4;
  padding: 1.5rem;
  border-radius: 8px;
  overflow-x: auto;
  box-shadow: 0 4px 12px rgba(0,0,0,0.1);
  border: 1px solid #393939;
}
.json-format-hint code {
  font-family: 'IBM Plex Mono', Consolas, Monaco, 'Andale Mono', 'Ubuntu Mono', monospace;
  font-size: 0.875rem;
  line-height: 1.5;
}
.error-notification {
  margin-bottom: 2rem;
  width: 100%;
  max-width: 500px;
}

/* QUIZ SCREEN */
.quiz-screen {
  width: 50vw;
  margin: 0 auto;
}
.quiz-header {
  display: flex;
  justify-content: space-between;
  align-items: flex-end;
  margin-bottom: 2rem;
  border-bottom: 1px solid #e0e0e0;
  padding-bottom: 1rem;
}
.quiz-header h2 {
  font-size: 1.5rem;
  font-weight: 400;
  margin-bottom: 0.5rem;
}
.badge {
  background-color: #0f62fe;
  color: white;
  padding: 0.25rem 0.75rem;
  border-radius: 50px;
  font-size: 0.75rem;
  font-weight: 600;
  letter-spacing: 0.3px;
  text-transform: uppercase;
}

.question-card {
  background: #ffffff;
  margin-bottom: 1.5rem;
  padding: 2rem;
  border-top: 3px solid transparent;
  transition: box-shadow 0.3s ease, border-color 0.3s ease;
  text-align: left;
  display: flex;
  flex-direction: column;
}
.question-card:hover {
  box-shadow: 0 2px 10px rgba(0,0,0,0.05);
}
.question-card.is-submitted {
  border-top-color: #e0e0e0;
}
.question-card.is-correct {
  border-top-color: #24a148;
}
.question-card.is-wrong {
  border-top-color: #da1e28;
}

.question-header {
  display: flex;
  gap: 1rem;
  align-items: flex-start;
  margin-bottom: 1.5rem;
}
.question-number {
  background: #f4f4f4;
  color: #0f62fe;
  font-weight: 600;
  min-width: 32px;
  height: 32px;
  display: flex;
  align-items: center;
  justify-content: center;
}
.question-text {
  font-size: 1.125rem;
  font-weight: 400;
  line-height: 1.4;
  margin-top: 0.25rem;
}
.options-container {
  padding-left: 3rem;
  display: flex;
  flex-direction: column;
  align-items: flex-start;
}
.options-container .bx--radio-button-group {
  display: flex;
  flex-direction: column;
  align-items: flex-start;
  gap: 1rem;
}

/* CUSTOMIZING CARBON RADIOS ON SUBMITTED STATE */
.correct-answer :deep(.bx--radio-button__label) {
  color: #24a148 !important;
  font-weight: 600;
}
.correct-answer :deep(.bx--radio-button__appearance) {
  border-color: #24a148 !important;
}
.correct-answer :deep(.bx--radio-button__appearance::before) {
  background-color: #24a148 !important;
}
.wrong-answer :deep(.bx--radio-button__label) {
  color: #da1e28 !important;
  text-decoration: line-through;
  opacity: 0.7;
}

.feedback-banner {
  margin-top: 1.5rem;
  padding-left: 3rem;
}
.feedback-content {
  display: flex;
  align-items: center;
  gap: 0.75rem;
  padding: 1rem;
  font-weight: 500;
}
.feedback-content.success {
  background-color: #defbe6;
  color: #198038;
  border-left: 3px solid #24a148;
}
.feedback-content.error {
  background-color: #fff1f1;
  color: #a2191f;
  border-left: 3px solid #da1e28;
}
.wrong-title {
  display: block;
  margin-bottom: 0.25rem;
}
.correct-is {
  font-weight: 400;
  color: #161616;
}

.navigation-section {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-top: 2rem;
}
.custom-nav-btn {
  background-color: transparent;
  color: #0f62fe;
  border: 1px solid #0f62fe;
  padding: 0.75rem 1.5rem;
  font-size: 1rem;
  font-weight: 500;
  cursor: pointer;
  display: inline-flex;
  align-items: center;
  gap: 0.5rem;
  transition: all 0.2s ease;
}
.custom-nav-btn:hover:not(:disabled) {
  background-color: #0f62fe;
  color: #ffffff;
}
.custom-nav-btn:disabled {
  border-color: #c6c6c6;
  color: #c6c6c6;
  cursor: not-allowed;
}
.progress-indicator {
  font-weight: 600;
  color: #525252;
}

.submit-section {
  display: flex;
  justify-content: flex-end;
  margin-top: 3rem;
}
.custom-submit-btn {
  background-color: #0f62fe;
  color: #ffffff;
  border: none;
  padding: 1rem 3rem;
  font-size: 1rem;
  font-weight: 500;
  cursor: pointer;
  display: inline-flex;
  align-items: center;
  gap: 0.75rem;
  transition: all 0.2s ease;
}
.custom-submit-btn:hover {
  background-color: #0353e9;
  padding-right: 2.5rem;
  padding-left: 3.5rem;
}

/* RESULT CARD */
.result-card {
  background: #ffffff;
  margin-top: 3rem;
  padding: 4rem 2rem;
  display: flex;
  flex-direction: column;
  align-items: center;
  text-align: center;
  border-top: 4px solid #0f62fe;
  box-shadow: 0 4px 16px rgba(0,0,0,0.1);
}
.score-ring {
  width: 150px;
  height: 150px;
  border-radius: 50%;
  border: 10px solid #f4f4f4;
  border-top-color: #0f62fe;
  border-right-color: #0f62fe;
  display: flex;
  align-items: center;
  justify-content: center;
  margin-bottom: 2rem;
  animation: spinRing 1s cubic-bezier(0.2, 0.8, 0.2, 1) forwards;
  transform: rotate(-45deg);
}
@keyframes spinRing {
  from { transform: rotate(-135deg); }
  to { transform: rotate(45deg); }
}
.score-percentage {
  font-size: 3rem;
  font-weight: 300;
  color: #161616;
  transform: rotate(-45deg);
}
.score-details h2 {
  font-size: 2rem;
  margin-bottom: 1rem;
  font-weight: 300;
}
.score-details p {
  font-size: 1.125rem;
  color: #525252;
  margin-bottom: 2rem;
}
.custom-outline-btn {
  background: transparent;
  color: #0f62fe;
  border: 1px solid #0f62fe;
  padding: 1rem 2rem;
  font-size: 1rem;
  font-weight: 500;
  cursor: pointer;
  display: inline-flex;
  align-items: center;
  gap: 0.75rem;
  transition: all 0.2s ease;
}
.custom-outline-btn:hover {
  background: #0f62fe;
  color: #ffffff;
}
</style>
