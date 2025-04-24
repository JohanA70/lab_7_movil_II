<template>
  <ion-page>
    <ion-header>
      <ion-toolbar>
        <ion-title>Escáner de QR</ion-title>
      </ion-toolbar>
    </ion-header>
    <ion-content class="ion-padding scanner-content">
      <ion-button @click="startScan" expand="block">
        Escanear Código QR
      </ion-button>

      <div v-if="isScanning" class="scanner-preview">
        <p class="scanner-instructions">Alinea el QR dentro de este marco</p>
      </div>

      <ion-list>
        <ion-list-header>Historial de Escaneos</ion-list-header>
        <ion-item v-for="(scan, index) in scanHistory" :key="index">
          <ion-label>{{ scan }}</ion-label>
        </ion-item>
      </ion-list>

      <div id="map" style="width: 100%; height: 400px; margin-top: 20px;"></div>
    </ion-content>
  </ion-page>
</template>

<script setup lang="ts">
import {
  IonContent,
  IonHeader,
  IonPage,
  IonTitle,
  IonToolbar,
  IonButton,
  IonList,
  IonListHeader,
  IonItem,
  IonLabel
} from '@ionic/vue';
import { ref } from 'vue';
import { CapacitorBarcodeScanner } from '@capacitor/barcode-scanner';
import { onMounted } from 'vue';
import 'mapbox-gl/dist/mapbox-gl.css';
import mapboxgl from 'mapbox-gl';

const scanHistory = ref<string[]>([]);
const isScanning = ref(false);
let mapInstance: mapboxgl.Map | null = null;
let currentMarker: mapboxgl.Marker | null = null;

const startScan = async () => {
  try {
    // Iniciar escaneo
    const result = await CapacitorBarcodeScanner.scanBarcode({ hint: 17 });
    console.log('Permiso de cámara:', result);
    console.log('Resultado del escaneo:', result.ScanResult);

    if (result.ScanResult) {
      const scanContent = result.ScanResult;

      try {
        const parsed = JSON.parse(scanContent);
        console.log('Contenido JSON escaneado:', parsed);

        scanHistory.value.push(JSON.stringify(parsed));

        alert(`Datos recibidos: ${JSON.stringify(parsed, null, 2)}`);
      } catch (e) {

        const inicio = scanContent.substring(0, 8);
        if (inicio.includes("located:")) {
          const coordinates = scanContent.substring(8, scanContent.length).split(',');
          const lat = parseFloat(coordinates[0]);
          const lng = parseFloat(coordinates[1]);

          if (!isNaN(lat) && !isNaN(lng)) {
            map(lat, lng);
            console.log('Mapa actualizado a coordenadas:', lat, lng);
          } else {
            console.warn('Coordenadas no válidas en el contenido escaneado');
          }
        } else if (scanContent.startsWith('http')) {
          window.open(scanContent, '_blank');
        } else if (scanContent.includes('@')) {
          window.location.href = `mailto:${scanContent}`;
        } else {
          alert(`Contenido escaneado: ${scanContent}`);
        }

        scanHistory.value.push(scanContent);
      }
    }
  } catch (error) {
    console.error('Error al escanear el código QR', error);
    alert('Error al escanear. Intente nuevamente.');
  }
};

mapboxgl.accessToken = 'pk.eyJ1Ijoiam9oYW5hbHZhcmV6IiwiYSI6ImNtOXVsOTI2dDBiZXQyanEzZjB1ejQ1bDYifQ.yF5ostspIOUKucl_c5yN_A';
onMounted(() => {
  map(-84.071, 9.9261);
});

function map(lng: number, lat: number) {
  if (!mapInstance) {
    mapInstance = new mapboxgl.Map({
      container: 'map',
      style: 'mapbox://styles/mapbox/streets-v12',
      center: [lng, lat],
      zoom: 9,
    });

    mapInstance.on('dblclick', (e) => {
      const { lng, lat } = e.lngLat;

      if (currentMarker) {
        currentMarker.remove();
      }

      currentMarker = new mapboxgl.Marker()
        .setLngLat([lng, lat])
        .addTo(mapInstance!);

      alert(`Coordenadas marcadas:\nLongitud: ${lng}\nLatitud: ${lat}`);
    });
  }

  mapInstance.setCenter([lng, lat]);

  if (currentMarker) {
    currentMarker.remove();
  }

  currentMarker = new mapboxgl.Marker()
    .setLngLat([lng, lat])
    .addTo(mapInstance);
}

</script>

<style scoped>
.scanner-content {
  --background: transparent;
  position: relative;
}

.scanner-preview {
  position: absolute;
  top: 120px;
  left: 50%;
  transform: translateX(-50%);
  width: 300px;
  height: 300px;
  border: 4px solid #00aaff;
  border-radius: 10px;
  box-shadow: 0 0 10px rgba(0, 0, 0, 0.5);
  overflow: hidden;
  z-index: 10;
}

.scanner-instructions {
  position: absolute;
  bottom: 0;
  width: 100%;
  margin: 0;
  padding: 5px 0;
  text-align: center;
  background: rgba(0, 0, 0, 0.5);
  color: #fff;
  font-size: 14px;
}
</style>
