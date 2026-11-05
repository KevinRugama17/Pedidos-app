<template>
  <ion-page>
    <ion-header>
      <ion-toolbar>
        <ion-title>Crear Post</ion-title>
      </ion-toolbar>
    </ion-header>
    <ion-content :fullscreen="true" class="ion-padding">
      <ion-card>
        <ion-card-header>
          <ion-card-title>Nuevo Post</ion-card-title>
          <ion-card-subtitle>Completa los datos o escanea un código QR</ion-card-subtitle>
        </ion-card-header>

        <ion-card-content>
          <ion-list>
            <ion-item>
              <ion-input
                v-model="title"
                label="Título"
                label-placement="floating"
                placeholder="Ingresa el título del post"
              ></ion-input>
            </ion-item>

            <ion-item>
              <ion-textarea
                v-model="body"
                label="Contenido (Body)"
                label-placement="floating"
                placeholder="Ingresa el contenido o escanea un QR"
                :rows="4"
              ></ion-textarea>
            </ion-item>

            <ion-item>
              <ion-select
                v-model="userId"
                label="Usuario"
                label-placement="floating"
                placeholder="Selecciona un usuario"
              >
                <ion-select-option
                  v-for="usuario in usuarios"
                  :key="usuario.id"
                  :value="usuario.id"
                >
                  {{ usuario.id }} - {{ usuario.name }}
                </ion-select-option>
              </ion-select>
            </ion-item>
          </ion-list>

          <div class="ion-margin-top">
            <ion-button expand="block" color="secondary" @click="escanearQR">
              <ion-icon slot="start" :icon="qrCodeOutline"></ion-icon>
              Escanear QR
            </ion-button>

            <ion-button
              expand="block"
              color="primary"
              class="ion-margin-top"
              :disabled="cargando || !title || !body || !userId"
              @click="crearPost"
            >
              <ion-icon slot="start" :icon="sendOutline"></ion-icon>
              {{ cargando ? 'Enviando...' : 'Crear Post' }}
            </ion-button>
          </div>
        </ion-card-content>
      </ion-card>

      <ion-card v-if="qrResultado">
        <ion-card-header>
          <ion-card-subtitle>Resultado del QR escaneado</ion-card-subtitle>
        </ion-card-header>
        <ion-card-content>
          <p>{{ qrResultado }}</p>
        </ion-card-content>
      </ion-card>

      <ion-card v-if="postCreado" color="light">
        <ion-card-header>
          <ion-card-title>Post Creado (ID: {{ postCreado.id }})</ion-card-title>
          <ion-card-subtitle>User ID: {{ postCreado.userId }}</ion-card-subtitle>
        </ion-card-header>
        <ion-card-content>
          <h3><strong>{{ postCreado.title }}</strong></h3>
          <p>{{ postCreado.body }}</p>
        </ion-card-content>
      </ion-card>
    </ion-content>
  </ion-page>
</template>

<script setup lang="ts">
import { ref } from 'vue';
import {
  IonPage,
  IonHeader,
  IonToolbar,
  IonTitle,
  IonContent,
  IonList,
  IonItem,
  IonInput,
  IonTextarea,
  IonSelect,
  IonSelectOption,
  IonButton,
  IonCard,
  IonCardHeader,
  IonCardTitle,
  IonCardSubtitle,
  IonCardContent,
  IonIcon,
} from '@ionic/vue';
import { qrCodeOutline, sendOutline } from 'ionicons/icons';
import {
  CapacitorBarcodeScanner,
  CapacitorBarcodeScannerTypeHint,
} from '@capacitor/barcode-scanner';
import { Posts } from '@/interfaces/posts';
import { Usuarios } from '@/interfaces/usuarios';

const usuarios = ref<Usuarios[]>([]);
const title = ref('');
const body = ref('');
const userId = ref<number | null>(null);
const qrResultado = ref('');
const cargando = ref(false);
const postCreado = ref<Posts | null>(null);

function cargarUsuarios() {
  fetch('https://jsonplaceholder.typicode.com/users')
    .then((response) => response.json())
    .then((json: Usuarios[]) => {
      usuarios.value = json;
      if (json.length > 0 && userId.value === null) {
        userId.value = json[0].id;
      }
    });
}

cargarUsuarios();

function rellenarCamposDesdeQR(textoQR: string) {
  const texto = textoQR.trim();

  // 1. Intentar parsear como JSON: {"title": "...", "body": "...", "userId": 1}
  try {
    const parsed = JSON.parse(texto);
    if (typeof parsed === 'object' && parsed !== null) {
      if (parsed.title !== undefined) title.value = String(parsed.title);
      if (parsed.body !== undefined) body.value = String(parsed.body);
      const id = Number(parsed.userId ?? parsed.userid ?? parsed.id);
      if (!Number.isNaN(id) && id >= 1 && id <= 10) {
        userId.value = id;
      }
      return;
    }
  } catch {
    // No es JSON, continuar con otros formatos
  }

  // 2. Intentar formato separado por "|", ";" o saltos de línea (ej: "Título|Contenido|3")
  const separadores = ['|', ';', '\n'];
  for (const sep of separadores) {
    if (texto.includes(sep)) {
      const partes = texto.split(sep).map((p) => p.trim());
      if (partes.length >= 2) {
        title.value = partes[0];
        body.value = partes[1];
        if (partes[2]) {
          const id = Number(partes[2]);
          if (!Number.isNaN(id) && id >= 1 && id <= 10) {
            userId.value = id;
          }
        }
        return;
      }
    }
  }

  // 3. Texto libre: rellena automáticamente título, contenido y asegura un usuario seleccionado
  title.value = texto.slice(0, 40);
  body.value = texto;
  if (!userId.value) {
    userId.value = 1;
  }
}

async function escanearQR() {
  try {
    const result = await CapacitorBarcodeScanner.scanBarcode({
      hint: CapacitorBarcodeScannerTypeHint.QR_CODE,
      scanInstructions: 'Apunta la cámara al código QR',
    });

    if (result.ScanResult) {
      qrResultado.value = result.ScanResult;
      rellenarCamposDesdeQR(result.ScanResult);
    }
  } catch (error) {
    console.error('Error al escanear QR:', error);
  }
}

function crearPost() {
  if (!userId.value) return;
  cargando.value = true;

  fetch('https://jsonplaceholder.typicode.com/posts', {
    method: 'POST',
    body: JSON.stringify({
      title: title.value,
      body: body.value,
      userId: userId.value,
    }),
    headers: {
      'Content-type': 'application/json; charset=UTF-8',
    },
  })
    .then((response) => response.json())
    .then((json: Posts) => {
      console.log(json);
      postCreado.value = json;
      title.value = '';
      body.value = '';
    })
    .finally(() => {
      cargando.value = false;
    });
}
</script>
