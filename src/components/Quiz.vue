<script setup lang="ts">
import { ref, onMounted, computed } from 'vue';
import Papa from 'papaparse';

interface Question {
  question: string;
  option1: string;
  option2: string;
  option3: string;
  option4: string;
  correctAnswer: string;
}

const questions = ref<Question[]>([]);
const currentQuestionIndex = ref(0);
const selectedAnswer = ref<string>('');
const userAnswers = ref<string[]>([]);
const showResults = ref(false);
const isLoading = ref(true);

onMounted(async () => {
  try {
    const response = await fetch('/quiz.csv');
    const csvText = await response.text();
    
    Papa.parse(csvText, {
      header: true,
      skipEmptyLines: true,
      complete: (results) => {
        const parsedQuestions = results.data.map((row: any) => ({
          question: row['Question'],
          option1: row['Option 1'],
          option2: row['Option 2'],
          option3: row['Option 3'],
          option4: row['Option 4'],
          correctAnswer: row['Correct Answer']
        }));
        
        // Shuffle the questions array
        questions.value = parsedQuestions.sort(() => Math.random() - 0.5);
        userAnswers.value = new Array(questions.value.length).fill('');
        isLoading.value = false;
      }
    });
  } catch (error) {
    console.error('Error loading quiz:', error);
    isLoading.value = false;
  }
});

const currentQuestion = computed(() => questions.value[currentQuestionIndex.value]);

const options = computed(() => {
  if (!currentQuestion.value) return [];
  return [
    currentQuestion.value.option1,
    currentQuestion.value.option2,
    currentQuestion.value.option3,
    currentQuestion.value.option4
  ];
});

const selectAnswer = (answer: string) => {
  selectedAnswer.value = answer;
};

const nextQuestion = () => {
  if (selectedAnswer.value) {
    userAnswers.value[currentQuestionIndex.value] = selectedAnswer.value;
    
    if (currentQuestionIndex.value < questions.value.length - 1) {
      currentQuestionIndex.value++;
      selectedAnswer.value = userAnswers.value[currentQuestionIndex.value] || '';
    } else {
      showResults.value = true;
    }
  }
};

const previousQuestion = () => {
  if (currentQuestionIndex.value > 0) {
    userAnswers.value[currentQuestionIndex.value] = selectedAnswer.value;
    currentQuestionIndex.value--;
    selectedAnswer.value = userAnswers.value[currentQuestionIndex.value] || '';
  }
};

const score = computed(() => {
  let correct = 0;
  questions.value.forEach((q, index) => {
    if (userAnswers.value[index] === q.correctAnswer) {
      correct++;
    }
  });
  return correct;
});

const percentage = computed(() => {
  return Math.round((score.value / questions.value.length) * 100);
});

const restartQuiz = () => {
  currentQuestionIndex.value = 0;
  selectedAnswer.value = '';
  userAnswers.value = new Array(questions.value.length).fill('');
  showResults.value = false;
};

const isCorrect = (questionIndex: number) => {
  const question = questions.value[questionIndex];
  return question && userAnswers.value[questionIndex] === question.correctAnswer;
};
</script>

<template>
  <div class="quiz-container">
    <div v-if="isLoading" class="loading">
      <p>Loading quiz...</p>
    </div>

    <div v-else-if="!showResults && questions.length > 0" class="quiz-content">
      <div class="quiz-header">
        <h1>Software Testing Quiz</h1>
        <div class="progress">
          Question {{ currentQuestionIndex + 1 }} of {{ questions.length }}
        </div>
        <div class="progress-bar">
          <div 
            class="progress-fill" 
            :style="{ width: ((currentQuestionIndex + 1) / questions.length * 100) + '%' }"
          ></div>
        </div>
      </div>

      <div class="question-section">
        <h2 class="question">{{ currentQuestion?.question }}</h2>
        
        <div class="options">
          <div 
            v-for="(option, index) in options" 
            :key="index"
            class="option"
            :class="{ selected: selectedAnswer === option }"
            @click="selectAnswer(option)"
          >
            <span class="option-label">{{ String.fromCharCode(65 + index) }}.</span>
            <span class="option-text">{{ option }}</span>
          </div>
        </div>

        <div class="navigation">
          <button 
            @click="previousQuestion" 
            :disabled="currentQuestionIndex === 0"
            class="btn btn-secondary"
          >
            Previous
          </button>
          <button 
            @click="nextQuestion" 
            :disabled="!selectedAnswer"
            class="btn btn-primary"
          >
            {{ currentQuestionIndex === questions.length - 1 ? 'Submit' : 'Next' }}
          </button>
        </div>
      </div>
    </div>

    <div v-else-if="showResults" class="results">
      <div class="results-header">
        <h1>Quiz Results</h1>
        <div class="score-card">
          <div class="score-number">{{ score }} / {{ questions.length }}</div>
          <div class="score-percentage">{{ percentage }}%</div>
          <div class="score-label">
            {{ percentage >= 80 ? 'Excellent!' : percentage >= 60 ? 'Good Job!' : 'Keep Practicing!' }}
          </div>
        </div>
      </div>

      <div class="answers-review">
        <h2>Review Your Answers</h2>
        <div v-for="(question, index) in questions" :key="index" class="answer-item">
          <div class="answer-header">
            <span class="question-number">Q{{ index + 1 }}</span>
            <span :class="['answer-status', isCorrect(index) ? 'correct' : 'incorrect']">
              {{ isCorrect(index) ? '✓ Correct' : '✗ Incorrect' }}
            </span>
          </div>
          <p class="review-question">{{ question.question }}</p>
          <p class="review-answer">
            <strong>Your answer:</strong> 
            <span :class="isCorrect(index) ? 'correct-text' : 'incorrect-text'">
              {{ userAnswers[index] || 'Not answered' }}
            </span>
          </p>
          <p v-if="!isCorrect(index)" class="review-answer">
            <strong>Correct answer:</strong> 
            <span class="correct-text">{{ question.correctAnswer }}</span>
          </p>
        </div>
      </div>

      <div class="results-actions">
        <button @click="restartQuiz" class="btn btn-primary">Restart Quiz</button>
      </div>
    </div>

    <div v-else class="error">
      <p>No questions available. Please check the CSV file.</p>
    </div>
  </div>
</template>

<style scoped>
.quiz-container {
  max-width: 800px;
  margin: 0 auto;
  padding: 20px;
  min-height: 100vh;
}

.loading, .error {
  text-align: center;
  padding: 40px;
  font-size: 18px;
  color: #2c3e50;
}

.quiz-header {
  margin-bottom: 30px;
}

.quiz-header h1 {
  margin: 0 0 15px 0;
  color: #2c3e50;
  font-size: 28px;
}

.progress {
  color: #666;
  margin-bottom: 10px;
  font-size: 14px;
}

.progress-bar {
  width: 100%;
  height: 8px;
  background-color: #e0e0e0;
  border-radius: 4px;
  overflow: hidden;
}

.progress-fill {
  height: 100%;
  background: linear-gradient(90deg, #42b883, #35495e);
  transition: width 0.3s ease;
}

.question-section {
  background: white;
  padding: 30px;
  border-radius: 12px;
  box-shadow: 0 2px 20px rgba(0, 0, 0, 0.08);
  border: 1px solid #e8e8e8;
}

.question {
  color: #2c3e50;
  margin: 0 0 25px 0;
  font-size: 20px;
  line-height: 1.5;
}

.options {
  display: flex;
  flex-direction: column;
  gap: 12px;
  margin-bottom: 30px;
}

.option {
  display: flex;
  align-items: center;
  padding: 15px 20px;
  border: 2px solid #e0e0e0;
  border-radius: 8px;
  cursor: pointer;
  transition: all 0.2s ease;
  background: white;
}

.option:hover {
  border-color: #42b883;
  background-color: #f0fdf4;
}

.option.selected {
  border-color: #42b883;
  background-color: #dcfce7;
  box-shadow: 0 0 0 3px rgba(66, 184, 131, 0.1);
}

.option-label {
  font-weight: bold;
  margin-right: 12px;
  color: #42b883;
  min-width: 24px;
}

.option-text {
  color: #2c3e50;
  flex: 1;
}

.navigation {
  display: flex;
  justify-content: space-between;
  gap: 10px;
}

.btn {
  padding: 12px 30px;
  border: none;
  border-radius: 6px;
  font-size: 16px;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.2s ease;
}

.btn-primary {
  background-color: #42b883;
  color: white;
}

.btn-primary:hover:not(:disabled) {
  background-color: #35a372;
}

.btn-secondary {
  background-color: #e0e0e0;
  color: #2c3e50;
}

.btn-secondary:hover:not(:disabled) {
  background-color: #d0d0d0;
}

.btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.results {
  background: white;
  padding: 30px;
  border-radius: 12px;
  box-shadow: 0 2px 20px rgba(0, 0, 0, 0.08);
  border: 1px solid #e8e8e8;
}

.results-header {
  text-align: center;
  margin-bottom: 40px;
}

.results-header h1 {
  margin: 0 0 20px 0;
  color: #2c3e50;
}

.score-card {
  background: linear-gradient(135deg, #42b883, #35495e);
  color: white;
  padding: 30px;
  border-radius: 12px;
  display: inline-block;
}

.score-number {
  font-size: 48px;
  font-weight: bold;
  margin-bottom: 10px;
}

.score-percentage {
  font-size: 32px;
  margin-bottom: 10px;
}

.score-label {
  font-size: 18px;
  opacity: 0.9;
}

.answers-review h2 {
  color: #2c3e50;
  margin-bottom: 20px;
}

.answer-item {
  background: #fafafa;
  padding: 20px;
  border-radius: 8px;
  margin-bottom: 15px;
  border-left: 4px solid #e0e0e0;
  border: 1px solid #f0f0f0;
  border-left: 4px solid #e0e0e0;
}

.answer-item.correct {
  border-left-color: #42b883;
}

.answer-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 10px;
}

.question-number {
  font-weight: bold;
  color: #2c3e50;
}

.answer-status {
  padding: 4px 12px;
  border-radius: 4px;
  font-size: 14px;
  font-weight: 500;
}

.answer-status.correct {
  background-color: #d4edda;
  color: #155724;
}

.answer-status.incorrect {
  background-color: #f8d7da;
  color: #721c24;
}

.review-question {
  color: #2c3e50;
  margin: 10px 0;
  font-weight: 500;
}

.review-answer {
  margin: 8px 0;
  color: #666;
}

.correct-text {
  color: #28a745;
  font-weight: 500;
}

.incorrect-text {
  color: #dc3545;
  font-weight: 500;
}

.results-actions {
  text-align: center;
  margin-top: 30px;
}

@media (max-width: 768px) {
  .quiz-container {
    padding: 10px;
  }

  .question-section, .results {
    padding: 20px;
  }

  .question {
    font-size: 18px;
  }

  .navigation {
    flex-direction: column;
  }

  .btn {
    width: 100%;
  }
}
</style>
