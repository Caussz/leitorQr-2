<script lang="ts" setup>
import { ref, onMounted } from 'vue';
import { BrowserQRCodeReader } from '@zxing/browser';

const qrCodeReader = new BrowserQRCodeReader();
const videoElement = ref<HTMLVideoElement | null>(null);
const cameras = ref<{ id: string; label: string }[]>([]);
const selectedCamera = ref<string | null>(null);
const qrCodeResult = ref<string | null>(null);
const cameraStarted = ref(false);
const loadingCamera = ref(false);
const errorMessage = ref<string | null>(null);
const scanHistory = ref<string[]>([]);

const listCameras = async () => {
  try {
    const devices = await navigator.mediaDevices.enumerateDevices();
    cameras.value = devices
      .filter((device) => device.kind === 'videoinput')
      .map((camera) => ({ id: camera.deviceId, label: camera.label || 'Câmera desconhecida' }));

    if (cameras.value.length > 0) {
      selectedCamera.value = cameras.value[0].id;
    } else {
      errorMessage.value = 'Nenhuma câmera encontrada.';
    }
  } catch (error) {
    errorMessage.value = 'Erro ao acessar câmeras: ' + error;
    console.error('Erro ao listar câmeras:', error);
  }
};

const startZxingScanner = async () => {
  try {
    if (!selectedCamera.value) {
      alert('Nenhuma câmera disponível para iniciar.');
      return;
    }

    qrCodeResult.value = null;
    cameraStarted.value = true;
    loadingCamera.value = true;
    errorMessage.value = null;

    const stream = await navigator.mediaDevices.getUserMedia({
      video: { deviceId: selectedCamera.value },
    });

    videoElement.value!.srcObject = stream;
    videoElement.value!.play();

    qrCodeReader.decodeFromVideoDevice(
      selectedCamera.value,
      videoElement.value!,
      (decodedResult) => {
        if (decodedResult) {
          console.log('Resultado do QR Code:', decodedResult);

          qrCodeResult.value = decodedResult.getText();
          scanHistory.value.push(qrCodeResult.value);
          stopZxingScanner();
        }
      }
    );
  } catch (error) {
    console.error('Erro ao iniciar o scanner:', error);
    errorMessage.value = 'Erro ao iniciar o scanner: ' + error;
    cameraStarted.value = false;
  } finally {
    loadingCamera.value = false;
  }
};

const stopZxingScanner = () => {
  if (videoElement.value) {
    const stream = videoElement.value.srcObject as MediaStream;
    const tracks = stream.getTracks();
    tracks.forEach((track) => track.stop());
    videoElement.value.srcObject = null;
  }
  cameraStarted.value = false;
};

const resetResult = () => {
  qrCodeResult.value = null;
  errorMessage.value = null;
};

const decodeContent = () => {
  if (qrCodeResult.value) {
    try {
      console.log('Conteúdo do QR Code:', qrCodeResult.value);

      const decodedBase64 = atob(qrCodeResult.value);
      console.log('Decodificado Base64:', decodedBase64);

      const jsonData = JSON.parse(decodedBase64);
      console.log('Dados JSON:', jsonData);
    } catch (error) {
      console.log('O conteúdo não é Base64 ou JSON válido:', qrCodeResult.value);
    }
  }
};

onMounted(() => {
  listCameras();
});
</script>

<template>
  <div class="container">
    <h1>Leitor de QR Code - ZXing</h1>

    <div v-if="!cameraStarted" class="camera-select">
      <label for="camera">Escolha a câmera:</label>
      <select id="camera" v-model="selectedCamera">
        <option v-for="camera in cameras" :key="camera.id" :value="camera.id">
          {{ camera.label }}
        </option>
      </select>
    </div>

    <div v-if="errorMessage" class="error-message">
      {{ errorMessage }}
    </div>

    <div class="buttons">
      <button v-if="!cameraStarted" @click="startZxingScanner" :disabled="!cameras.length || loadingCamera">
        {{ loadingCamera ? 'Carregando...' : cameras.length ? 'Iniciar Scanner' : 'Nenhuma câmera disponível' }}
      </button>
      <button v-if="cameraStarted" @click="stopZxingScanner" class="stop-button">
        Parar Scanner
      </button>
      <button v-if="qrCodeResult" @click="resetResult" class="reset-button">
        Resetar Resultado
      </button>
    </div>

    <video ref="videoElement" class="scanner"></video>

    <div v-if="qrCodeResult" class="result">
      <h2>Resultado do QR Code:</h2>
      <p>{{ qrCodeResult }}</p>

      <div>
        <h3>Decodificação (se aplicável):</h3>
        <button @click="decodeContent">Decodificar Conteúdo</button>
      </div>
    </div>

    <div v-if="scanHistory.length" class="history">
      <h3>Histórico de Leitura:</h3>
      <ul>
        <li v-for="(result, index) in scanHistory" :key="index">{{ result }}</li>
      </ul>
    </div>
  </div>
</template>

<style scoped>
.container {
  max-width: 600px;
  margin: 0 auto;
  padding: 1.5em;
  text-align: center;
  font-family: 'Roboto', sans-serif;
  color: #333;
  background-color: #f7f9fc;
  border-radius: 8px;
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.1);
}

.camera-select {
  margin: 1.5em 0;
}

label {
  font-weight: bold;
  margin-right: 0.5em;
}

select {
  padding: 0.5em;
  border: 1px solid #ccc;
  border-radius: 4px;
}

.buttons {
  margin: 1.5em 0;
}

button {
  padding: 0.7em 1.5em;
  margin: 0.5em;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  font-size: 1em;
  transition: background-color 0.3s ease;
}

button:hover {
  background-color: #eaeaea;
}

button:disabled {
  background-color: #ccc;
  cursor: not-allowed;
}

.stop-button {
  background-color: #ff4d4d;
  color: white;
}

.stop-button:hover {
  background-color: #ff1a1a;
}

.reset-button {
  background-color: #ffa31a;
  color: white;
}

.reset-button:hover {
  background-color: #ff8000;
}

.scanner {
  width: 100%;
  max-width: 400px;
  border: 2px solid #ccc;
  border-radius: 8px;
  margin: 1.5em auto;
}

.result {
  background-color: #fff;
  padding: 1.5em;
  border: 1px solid #ddd;
  border-radius: 8px;
  margin-top: 1.5em;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

.history {
  margin-top: 1.5em;
  text-align: left;
}
</style>
