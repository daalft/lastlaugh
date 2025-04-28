<template>
  <div v-if="hashPresent" class="p-4">
    <div v-if="submissionSuccess">
      <div class="max-w-2xl mx-auto bg-green-100 p-6 rounded shadow text-center">
        <h2 class="text-2xl font-bold text-green-700 mb-4">Thank you for your submission!</h2>
        <p class="text-green-800 mb-6">Your comic panel has been successfully uploaded. We appreciate your contribution!</p>
        <button @click="resetForm" class="px-4 py-2 bg-blue-500 rounded">
          Submit Another Panel
        </button>
      </div>
    </div>

    <div v-else>
      <div v-if="!agreed">
        <div class="max-w-3xl mx-auto bg-white p-6 rounded shadow">
          <h1 class="text-2xl font-bold mb-4">Contributor License, Consent & Prize Terms</h1>
          <div class="text-sm space-y-4" v-html="legalText"></div>

          <div class="mt-6">
            <label>
              <input type="checkbox" v-model="consentGiven" class="mr-2" />
              I have read and agree to the above terms, and I grant permission for my submission to be included in the public dataset under CC BY 4.0.
            </label>
          </div>

          <button
            :disabled="!consentGiven"
            @click="agreed = true"
            class="mt-4 px-4 py-2 bg-blue-500 rounded disabled:opacity-50"
          >
            Proceed to Upload
          </button>
        </div>
      </div>

      <div v-else>
        <div class="max-w-3xl mx-auto bg-white p-6 rounded shadow">
          <h2 class="text-xl font-bold mb-4">Upload Your Comic Panel</h2>

          

          <form @submit.prevent="confirmSubmit">
            <div
              class="mb-4 border-2 border-dashed border-gray-300 p-6 text-center cursor-pointer"
              @dragover.prevent
              @drop.prevent="handleDrop"
              @click="fileInput.click()"
            >
              <p v-if="!selectedFile">Drag and drop your panel image here, or click to select a file</p>
              <div v-else>
                <p>Selected: {{ selectedFile.name }}</p>
                <img :src="previewUrl" alt="Preview" class="mt-4 max-h-64 max-w-xs mx-auto object-contain" />
                <button type="button" @click.stop="clearFile" class="mt-2 px-3 py-1 bg-red-500 rounded">
                  Remove File
                </button>
              </div>
              <input
                type="file"
                accept="image/png, image/jpeg"
                ref="fileInput"
                style="display: none"
                @change="handleFile"
              />
            </div>

            <div class="mb-4">
              <label for="description" class="block mb-2 text-left">Select which template your submission is for
                <select v-model="template" class="border p-2 w-full">
                  <option value="" disabled>Select a template</option>
                  <option value="comicHorse">Horse</option>
                  <option value="comicOffice">Office Mistake</option>
                  <option value="comicProfessor">Professor</option>
                  </select>
              </label>
            </div>
            <div class="mb-4">
              <label for="description" class="block mb-2 text-left">Description (optional)
                <textarea v-model="description" class="border p-2 w-full" placeholder="Describe your panel and why it is humorous (optional)"></textarea>
              </label>
            </div>

            <div class="mb-4">
              <label for="email" class="block mb-2 text-left">Email (optional)
              
              <input type="email" v-model="email" class="border p-2 w-full" placeholder="Your email (if you want to be eligible for a prize)"/>
              <span class="text-sm text-gray-500">Leave blank if not submitting for a prize</span>
            </label>
            </div>
<!--
            <div class="mb-4">
              <label>
                <input type="checkbox" v-model="creditConsent" class="mr-2" />
                I would like to be credited as a contributor.
              </label>
            </div>
-->
            <div class="mb-4">
              <label for="creditName" class="block mb-2 text-left">
                Name (optional)
                <input type="text" v-model="creditName" class="border p-2 w-full" placeholder="Your name (if you want to be credited)" />
                <span class="text-sm text-gray-500">Leave blank for anonymous submission</span>
              </label>
            </div>

            <div class="mb-3 border-2 border-dashed border-gray-300 p-4 text-center">
            <label>
              <input type="checkbox" v-model="isHumanChecked" />
              I am not a robot
            </label>
          </div>

            <button
              type="submit"
              class="px-4 py-2 bg-green-500 rounded flex items-center justify-center"
              :disabled="submitting"
            >
              <span v-if="submitting" class="loader mr-2"></span>
              <span>{{ submitting ? 'Submitting...' : 'Submit' }}</span>
            </button>
          </form>
        </div>
      </div>
    </div>
  </div>

  <div v-else class="p-4 text-center">
    <p class="text-lg">Missing or invalid link. Please use a valid invitation URL.</p>
  </div>

  <div v-if="showModal" class="fixed inset-0 backdrop-blur-xs bg-opacity-50 flex items-center justify-center z-50">
    <div class="bg-white p-6 rounded shadow max-w-sm w-full text-center relative z-50">
      <h2 class="text-xl font-bold mb-4">Confirm Submission</h2>
      <p class="mb-6">Are you sure you want to submit this panel?</p>
      <div class="flex justify-center space-x-4">
        <button @click="handleSubmit" class="px-4 py-2 bg-green-500 rounded">Yes</button>
        <button @click="showModal = false" class="px-4 py-2 bg-gray-300 rounded">Cancel</button>
      </div>
    </div>
  </div>

  <div v-if="snackbarMessage" class="fixed bottom-4 left-1/2 transform -translate-x-1/2 bg-red-500 text-white px-6 py-3 rounded shadow z-50">
    {{ snackbarMessage }}
  </div>
</template>

<script setup>
import { ref, onMounted, nextTick } from 'vue'

const agreed = ref(false)
const consentGiven = ref(false)
const isHumanChecked = ref(false)
const email = ref('')
const selectedFile = ref(null)
const previewUrl = ref('')
const creditConsent = ref(false)
const creditName = ref('')
const hashPresent = ref(false)
const submissionSuccess = ref(false)
const submitting = ref(false)
const showModal = ref(false)
const snackbarMessage = ref('')
const fileInput = ref(null)

const legalText = `  <div class="max-w-4xl mx-auto bg-white p-8 rounded-2xl shadow-sm space-y-8 text-left">

    <h2 class="text-3xl font-bold">SemEval LAUGH Shared Task – Comic Panel Completion</h2>

    <p>Thank you for contributing to the SemEval LAUGH shared task! Please read the following terms carefully before submitting. By participating, you agree to the terms and conditions below.</p>

    <hr class="border-gray-300">

    <section>
      <h2 class="text-2xl font-semibold mb-2">1. Your Contribution</h2>
      <p>You are submitting an original, human-authored comic panel (the "Contribution") to complete a four-panel comic in which the second panel is missing.</p>
    </section>

    <hr class="border-gray-300">

    <section>
      <h2 class="text-2xl font-semibold mb-2">2. Copyright and License</h2>
      <p>You retain copyright over your Contribution.</p>
      <p class="mt-2">By submitting, you grant the SemEval LAUGH organizers a <strong>non-exclusive, worldwide, royalty-free, and irrevocable license</strong> to use, reproduce, modify, adapt, and redistribute your Contribution for research purposes. The resulting dataset, including your submission, will be released under a <strong>Creative Commons Attribution 4.0 International (CC BY 4.0)</strong> license.</p>
    </section>

    <hr class="border-gray-300">

    <section>
      <h2 class="text-2xl font-semibold mb-2">3. Attribution Options</h2>
      <p>You may choose whether to submit your Contribution <strong>anonymously</strong> or <strong>with attribution</strong>.</p>
      <ul class="list-disc list-inside mt-2 space-y-1">
        <li>If you choose attribution, your name (and optionally your affiliation) will appear in a <strong>collective contributors list</strong> (e.g., a <code>credits.txt</code> file or dataset documentation). Individual contributions will not be directly linked to specific names.</li>
        <li>If you choose anonymity, your identity will not be associated with your submission in any public materials.</li>
      </ul>
    </section>

    <hr class="border-gray-300">

    <section>
      <h2 class="text-2xl font-semibold mb-2">4. Prize Terms</h2>
      <p>Prizes will be awarded to the <strong>top three submissions</strong> based on creativity and relevance, as determined by the SemEval LAUGH organizers:</p>
      <ul class="list-disc list-inside mt-2 space-y-1">
        <li>🥇 1st place: $100 Amazon gift card</li>
        <li>🥈 2nd place: $50 Amazon gift card</li>
        <li>🥉 3rd place: $10 Amazon gift card</li>
      </ul>

      <p class="mt-4">To be eligible for a prize:</p>
      <ul class="list-disc list-inside mt-2 space-y-1">
        <li>You must provide a <strong>valid email address</strong> in the submission form.</li>
        <li>Anonymous submissions <strong>without email</strong> are <strong>not eligible for prizes</strong>.</li>
        <li>Participants are responsible for ensuring they are <strong>legally allowed to receive prizes</strong> under local laws.</li>
        <li>The organizers reserve the right to <strong>withhold prizes</strong> if:
          <ul class="list-disc list-inside ml-6 space-y-1">
            <li>Submissions do not meet quality expectations;</li>
            <li>Fewer than three qualifying submissions are received;</li>
            <li>Eligibility requirements (e.g., originality, legal status, identity) are not met;</li>
            <li>Legal, logistical, or budgetary constraints arise.</li>
          </ul>
        </li>
      </ul>

      <p class="mt-4">The organizers are not responsible for:</p>
      <ul class="list-disc list-inside mt-2 space-y-1">
        <li>Lost or undeliverable prizes,</li>
        <li>Currency conversion fees,</li>
        <li>Tax obligations in the recipient's jurisdiction.</li>
      </ul>
    </section>

    <hr class="border-gray-300">

    <section>
      <h2 class="text-2xl font-semibold mb-2">5. Data Privacy (GDPR Compliance)</h2>
      <p>Your personal data (name, email, IP address if applicable) will be collected and processed for the following purposes:</p>
      <ul class="list-disc list-inside mt-2 space-y-1">
        <li>Managing submissions and attribution (if selected),</li>
        <li>Contacting prize winners,</li>
        <li>Ensuring research reproducibility.</li>
      </ul>

      <p class="mt-4">Only the SemEval LAUGH organizers will access this data. Your data will not be shared with third parties and will be retained for up to <strong>two years</strong> after the conclusion of the shared task.</p>

      <p class="mt-4">Under the <strong>General Data Protection Regulation (GDPR)</strong>, you have the right to:</p>
      <ul class="list-disc list-inside mt-2 space-y-1">
        <li>Access, correct, or delete your personal data,</li>
        <li>Withdraw your consent at any time,</li>
        <li>Request anonymization or exclusion from attribution lists.</li>
      </ul>

      <p class="mt-4">To make such a request, contact us at: <strong>david.alfter@gu.se</strong>, <strong>mattias.appelgren@gu.se</strong></p>
    </section>

    <hr class="border-gray-300">

    <section>
      <h2 class="text-2xl font-semibold mb-2">6. Legal Jurisdiction and Liability</h2>
      <p>This agreement is governed by the laws of <strong>Sweden</strong>, without regard to its conflict of law provisions.</p>
      <p class="mt-2">Any disputes arising in connection with this agreement shall be subject to the exclusive jurisdiction of the <strong>Swedish courts</strong>.</p>

      <p class="mt-4">By submitting a Contribution:</p>
      <ul class="list-disc list-inside mt-2 space-y-1">
        <li>You agree to comply with all applicable laws and regulations.</li>
        <li>You confirm your submission is your own original work, free of copyright infringement, personal data, or offensive content.</li>
        <li>You release the organizers from liability for any damages arising from your participation or use of the dataset, including issues related to attribution, licensing, or prize delivery.</li>
      </ul>
    </section>
<!--
    <hr class="border-gray-300">

    <section>
      <h2 class="text-2xl font-semibold mb-4">✅ Consent</h2>

      <div class="flex items-start space-x-3">
        <input id="consent" type="checkbox" class="mt-1 w-5 h-5 text-blue-600 border-gray-300 rounded">
        <label for="consent" class="text-gray-700">
          I have read and agree to the above terms, and I grant permission for my submission to be included in the public dataset under CC BY 4.0.
        </label>
      </div>
    </section>
-->
  </div>`

function handleFile(event) { const file = event.target.files[0]; validateAndSetFile(file); }
function handleDrop(event) { const file = event.dataTransfer.files[0]; validateAndSetFile(file); }
function validateAndSetFile(file) {
  if (!file) return
  const validTypes = ['image/jpeg', 'image/png']
  const maxSize = 5 * 1024 * 1024
  if (!validTypes.includes(file.type)) { showError('Only JPEG and PNG images are allowed.'); return }
  if (file.size > maxSize) { showError('File size must be less than 5MB.'); return }
  selectedFile.value = file
  previewUrl.value = URL.createObjectURL(file)
}
function clearFile() { selectedFile.value = null; previewUrl.value = '' }
function resetForm() {
  agreed.value = true
  consentGiven.value = false
  isHumanChecked.value = false
  email.value = ''
  selectedFile.value = null
  previewUrl.value = ''
  creditConsent.value = false
  submissionSuccess.value = false
  submitting.value = false
  nextTick(() => { window.scrollTo({ top: 0, behavior: 'smooth' }) })
}
function confirmSubmit() {
  if (isHumanChecked.value) { return }
  if (!selectedFile.value) { showError('Please select a valid file.'); return }
  if (email.value && !validateEmail(email.value)) { showError('Please provide a valid email address.'); return }
  showModal.value = true
}
function validateEmail(email) {
  const re = /^[^\s@]+@[^\s@]+\.[^\s@]+$/
  return re.test(email)
}
/*
async function handleSubmit() {
  showModal.value = false;
  submitting.value = true;

  if (!selectedFile.value) {
    showError('Please select a file before submitting.');
    submitting.value = false;
    return;
  }

  try {
    const formData = new FormData();
    formData.append('file', selectedFile.value);
    formData.append('email', email.value);
    formData.append('creditConsent', creditConsent.value ? 'yes' : 'no');
    console.table('Submitting form data:', {
      file: selectedFile.value,
      email: email.value,
      creditConsent: creditConsent.value,
    });
    const response = await fetch('http://localhost:9200/submit', {
      method: 'POST',
      body: formData,
    });

    if (!response.ok) {
      throw new Error('Server responded with an error');
    }

    const result = await response.json();

    if (result.success) {
      submissionSuccess.value = true;
      nextTick(() => {
        window.scrollTo({ top: 0, behavior: 'smooth' });
      });
    } else {
      throw new Error(result.message || 'Unknown server error');
    }

  } catch (error) {
    console.error('Submission failed:', error);
    showError('Submission failed. Please try again.');
  } finally {
    submitting.value = false;
  }
}*/
/*FAKE SUBMIT*/
function handleSubmit() {
  showModal.value = false
  submitting.value = true
  setTimeout(() => {
    const success = Math.random() > 0.1
    if (success) {
      submissionSuccess.value = true
      nextTick(() => { window.scrollTo({ top: 0, behavior: 'smooth' }) })
    } else {
      showError('Submission failed. Please try again.')
    }
    submitting.value = false
  }, 1500)
}
/*END FAKE SUBMIT*/
function showError(message) {
  snackbarMessage.value = message
  setTimeout(() => { snackbarMessage.value = '' }, 3000)
}
onMounted(() => {
  const url = new URL(window.location.href)
  if (url.searchParams.get('hash')) {
    hashPresent.value = true
  }
})
</script>

<style>
body { background: #f0f4f8; }
.loader {
  border: 2px solid #f3f3f3;
  border-top: 2px solid #3498db;
  border-radius: 50%;
  width: 16px;
  height: 16px;
  animation: spin 1s linear infinite;
}
@keyframes spin {
  0% { transform: rotate(0deg); }
  100% { transform: rotate(360deg); }
}
.mb-3 {
  display: none;
}
</style>
