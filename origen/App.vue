<script setup>
import { ref, onMounted } from 'vue';

const backendUrl = "PASTE_YOUR_BACKEND_URL_HERE"; // We will fill this in later
const briefing = ref(null);
const conversationHistory = ref([]);
const isListening = ref(false);
const activeTab = ref('briefing');

onMounted(async () => {
  try {
    const response = await fetch(`${backendUrl}/api/daily-briefing`);
    if (!response.ok) throw new Error(`Error: ${response.status}`);
    briefing.value = await response.json();
    speakResponse("Buenos días. Tu briefing diario está listo.");
  } catch (error) {
    console.error("Could not fetch briefing:", error);
    speakResponse("Error al conectar con el cerebro de Kórima.");
  }
});

function startListening() {
  const SpeechRecognition = window.SpeechRecognition || window.webkitSpeechRecognition;
  if (!SpeechRecognition) { alert("Sorry, your browser doesn't support speech recognition."); return; }
  const recognition = new SpeechRecognition();
  recognition.lang = 'es-ES';
  isListening.value = true;
  recognition.onresult = async (event) => { const transcript = event.results[0][0].transcript; await sendTranscriptToBrain(transcript); };
  recognition.onerror = (event) => { console.error("Speech recognition error:", event.error); isListening.value = false; };
  recognition.onend = () => { isListening.value = false; };
  recognition.start();
}

function speakResponse(textToSpeak) {
  if (!textToSpeak || !window.speechSynthesis) return;
  const utterance = new SpeechSynthesisUtterance(textToSpeak);
  utterance.lang = 'es-ES';
  window.speechSynthesis.speak(utterance);
}

async function sendTranscriptToBrain(text) {
  try {
    const response = await fetch(`${backendUrl}/api/conversation`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ text: text })
    });
    if (!response.ok) throw new Error("The brain could not process the memory.");
    const data = await response.json();
    speakResponse(data.response_text);
    conversationHistory.value.unshift({ user: text, korima: data.response_text });
    activeTab.value = 'memory';
  } catch (error) {
    console.error("Error sending transcript to brain:", error);
    speakResponse("I had a problem trying to remember that.");
  }
}
</script>
<template>
  <div class="app-container">
    <header class="app-header"><img src="https://i.imgur.com/gJeEwjg.png" alt="Kórima Logo" class="logo"><h1>Kórima</h1></header>
    <main class="app-main">
      <div class="mic-container"><button @click="startListening" :class="{ 'listening': isListening }" class="mic-button" title="Presiona para hablar">🎤</button><p v-if="isListening">Escuchando...</p></div>
      <nav class="tabs"><button :class="{ active: activeTab === 'briefing' }" @click="activeTab = 'briefing'">Briefing</button><button :class="{ active: activeTab === 'memory' }" @click="activeTab = 'memory'">Memoria</button><button :class="{ active: activeTab === 'about' }" @click="activeTab = 'about'">Acerca De</button></nav>
      <div class="tab-content">
        <div v-if="activeTab === 'briefing'">
          <section v-if="briefing" class="card">
            <h2>🎯 Briefing del {{ new Date().toLocaleDateString('es-ES', { dateStyle: 'long' }) }}</h2>
            <p v-if="briefing.error">{{ briefing.error }}</p>
            <div v-else><p>{{ briefing.focus.description }}</p><ul v-if="briefing.focus.events"><li v-for="event in briefing.focus.events" :key="event">{{ event }}</li></ul></div>
          </section>
          <div v-else class="card"><p>Cargando briefing desde el cerebro de Kórima...</p></div>
        </div>
        <div v-if="activeTab === 'memory'">
          <section class="card conversation-history">
            <h2>Memoria Reciente</h2>
            <div v-if="conversationHistory.length > 0"><ul><li v-for="(entry, index) in conversationHistory" :key="index"><p><strong>Tú:</strong> {{ entry.user }}</p><p><strong>Kórima:</strong> {{ entry.korima }}</p></li></ul></div>
            <p v-else>Aún no hemos conversado. Presiona el micrófono para crear el primer recuerdo.</p>
          </section>
        </div>
        <div v-if="activeTab === 'about'">
           <section class="card">
            <h2>Acerca de Kórima v3.0</h2>
            <p>Esta es una aplicación de IA multimodal y proactiva, construida desde cero por ti.</p>
            <p><strong>Frontend:</strong> Desplegado en Vercel.</p><p><strong>Backend:</strong> Desplegado en Google Cloud Run.</p><p><strong>Memoria:</strong> Almacenada en Google Firestore.</p>
          </section>
        </div>
      </div>
    </main>
  </div>
</template>
<style>
@import url('https://fonts.googleapis.com/css2?family=Inter:wght@400;600;700&display=swap');
:root { --background-color: #f0f2f5; --card-background: #ffffff; --text-primary: #1c1e21; --text-secondary: #606770; --accent-color: #1877f2; --border-color: #dddfe2; --shadow: 0 4px 12px rgba(0,0,0,0.1); }
body { margin: 0; font-family: 'Inter', sans-serif; background-color: var(--background-color); color: var(--text-primary); }
.app-container { max-width: 768px; margin: 2rem auto; padding: 1rem; }
.app-header { display: flex; align-items: center; justify-content: center; gap: 1rem; margin-bottom: 2rem; }
.logo { width: 50px; height: 50px; }
h1 { font-size: 2.5rem; font-weight: 700; }
.card { background-color: var(--card-background); border-radius: 12px; padding: 1.5rem; margin-bottom: 1.5rem; box-shadow: var(--shadow); border: 1px solid var(--border-color); }
h2 { border-bottom: 1px solid var(--border-color); padding-bottom: 0.75rem; margin-top: 0; font-size: 1.25rem; color: var(--text-secondary); }
.mic-container { text-align: center; margin-bottom: 2rem; }
.mic-button { background-color: #fff; border: 1px solid var(--border-color); border-radius: 50%; width: 70px; height: 70px; font-size: 32px; cursor: pointer; transition: all 0.2s ease; box-shadow: 0 2px 4px rgba(0,0,0,0.1); }
.mic-button:hover { transform: scale(1.1); box-shadow: 0 4px 8px rgba(0,0,0,0.15); }
.mic-button.listening { background-color: #ff4136; color: white; animation: pulse 1.5s infinite; }
.tabs { display: flex; margin-bottom: 1.5rem; border-bottom: 1px solid var(--border-color); }
.tabs button { padding: 1rem 1.5rem; border: none; background: none; cursor: pointer; font-size: 1rem; font-weight: 600; color: var(--text-secondary); position: relative; border-bottom: 3px solid transparent; }
.tabs button.active { color: var(--accent-color); border-bottom-color: var(--accent-color); }
.conversation-history ul { list-style: none; padding: 0; margin: 0; }
.conversation-history li { border-bottom: 1px solid #eee; padding: 1rem 0; }
.conversation-history li:last-child { border-bottom: none; }
.conversation-history p { margin: 0.5rem 0; }
@keyframes pulse { 0% { box-shadow: 0 0 0 0 rgba(255, 65, 54, 0.7); } 70% { box-shadow: 0 0 0 20px rgba(255, 65, 54, 0); } 100% { box-shadow: 0 0 0 0 rgba(255, 65, 54, 0); } }
</style>
