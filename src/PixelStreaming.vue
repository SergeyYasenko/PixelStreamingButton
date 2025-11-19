<template>
   <div class="pixel-streaming-container">
      <div class="video-wrapper">
         <div ref="videoContainer" class="video-container"></div>

         <div v-if="!isConnected" class="overlay">
            <div class="connection-panel">
               <h2>Pixel Streaming</h2>
               <div class="input-group">
                  <label>Signalling Server URL:</label>
                  <input
                     v-model="signallingUrl"
                     type="text"
                     placeholder="ws://localhost:80"
                     :disabled="isConnecting"
                  />
               </div>
               <button
                  @click="connect"
                  :disabled="isConnecting"
                  class="connect-btn"
               >
                  {{ isConnecting ? "Подключение..." : "Подключиться" }}
               </button>
               <div v-if="errorMessage" class="error-message">
                  {{ errorMessage }}
               </div>
            </div>
         </div>

         <ConnectedDisplay
            v-if="isConnected"
            :last-message="lastMessage"
            :received-messages="receivedMessages"
            @send-data="sendJsonData"
            @send-to-engine="sendToEngine"
         />
      </div>
   </div>
</template>

<script setup>
import { ref, onBeforeUnmount } from "vue";
import {
   PixelStreaming,
   Config,
} from "@epicgames-ps/lib-pixelstreamingfrontend-ue5.5";
import ConnectedDisplay from "./ConnectedDisplay.vue";

const videoContainer = ref(null);
const signallingUrl = ref("ws://localhost:80");
const isConnected = ref(false);
const isConnecting = ref(false);
const errorMessage = ref("");
const receivedMessages = ref([]);
const lastMessage = ref("");

let pixelStreaming = null;

// Функция для инвертирования координат мыши по оси X
const invertMouseCoordinates = (event) => {
   const target = event.target;
   const rect = target.getBoundingClientRect();
   const centerX = rect.width / 2;
   const relativeX = event.clientX - rect.left;
   const invertedX = rect.width - relativeX;

   return {
      clientX: rect.left + invertedX,
      clientY: event.clientY,
   };
};

// Перехват событий мыши для инвертирования координат
const setupMouseInterception = () => {
   const observer = new MutationObserver(() => {
      const video = videoContainer.value?.querySelector("video");
      if (video && !video.dataset.intercepted) {
         video.dataset.intercepted = "true";

         // Перехватываем все события мыши
         const events = [
            "mousedown",
            "mouseup",
            "mousemove",
            "click",
            "dblclick",
         ];

         events.forEach((eventType) => {
            video.addEventListener(
               eventType,
               (e) => {
                  // Пропускаем уже обработанные события
                  if (e.inverted) {
                     return;
                  }

                  // Останавливаем оригинальное событие
                  e.preventDefault();
                  e.stopImmediatePropagation();

                  const inverted = invertMouseCoordinates(e);

                  // Создаем новое событие с инвертированными координатами
                  const newEvent = new MouseEvent(eventType, {
                     bubbles: e.bubbles,
                     cancelable: e.cancelable,
                     view: e.view,
                     detail: e.detail,
                     screenX: e.screenX,
                     screenY: e.screenY,
                     clientX: inverted.clientX,
                     clientY: inverted.clientY,
                     ctrlKey: e.ctrlKey,
                     altKey: e.altKey,
                     shiftKey: e.shiftKey,
                     metaKey: e.metaKey,
                     button: e.button,
                     buttons: e.buttons,
                     relatedTarget: e.relatedTarget,
                  });

                  // Помечаем событие как обработанное
                  newEvent.inverted = true;

                  // Отправляем новое событие
                  video.dispatchEvent(newEvent);
               },
               true
            ); // Используем capture фазу
         });

         observer.disconnect();
      }
   });

   if (videoContainer.value) {
      observer.observe(videoContainer.value, {
         childList: true,
         subtree: true,
      });
   }
};

const connect = async () => {
   if (!signallingUrl.value) {
      errorMessage.value = "Пожалуйста, введите URL сервера";
      return;
   }

   try {
      isConnecting.value = true;
      errorMessage.value = "";

      const config = new Config({
         initialSettings: {
            // ============================================
            // ОСНОВНЫЕ НАСТРОЙКИ ПОДКЛЮЧЕНИЯ
            // ============================================
            ss: signallingUrl.value, // URL сигнального сервера (WebSocket)
            AutoPlayVideo: true, // Автоматически воспроизводить видео при подключении (default: false)
            AutoConnect: true, // Автоматически подключаться при загрузке (default: false)
            ss_autoconnect: true, // Альтернативный флаг автоподключения (default: false)

            // ============================================
            // НАСТРОЙКИ ВИДЕО И АУДИО
            // ============================================
            StartVideoMuted: false, // Запускать видео без звука (default: false)
            HoveringMouse: false, // Отображать курсор мыши при наведении (default: true)
            FakeMouseWithTouches: true, // Эмулировать мышь через тач-события (default: true)

            // Использование аудио/микрофона
            UseMic: false, // Использовать микрофон (default: false)
            UseAudio: false, // Использовать аудио из стрима (default: false)

            // ============================================
            // НАСТРОЙКИ ЛОГИРОВАНИЯ
            // ============================================
            // Уровни: "Verbose", "Info", "Warning", "Error", "None"
            LogLevel: "Error", // Минимальное логирование для production (default: "Info")

            // ============================================
            // НАСТРОЙКИ ВИДЕО КОДЕКА
            // ============================================
            PreferH264: true, // Предпочитать H.264 (лучшая совместимость) (default: false)
            ForceH264: false, // Принудительно использовать H.264 (default: false)
            PreferVP8: false, // Предпочитать VP8 (лучше для некоторых браузеров) (default: false)
            PreferVP9: false, // Предпочитать VP9 (более эффективный, но меньше поддержка) (default: false)

            // ============================================
            // НАСТРОЙКИ СЕТИ (TURN/STUN)
            // ============================================
            ForceTURN: false, // Принудительно использовать TURN сервер (default: false)
            ForceMONO: false, // Принудительно использовать моно аудио (default: false)

            // ============================================
            // НАСТРОЙКИ БИТРЕЙТА (в битах в секунду)
            // ============================================
            MinBitrate: 100, // Минимальный битрейт 500kbps (default: 100000)
            MaxBitrate: 100000, // Максимальный битрейт 8Mbps (default: 20000000)
            LowBitrateTimeoutMs: 1000, // Таймаут низкого битрейта 10 сек (default: 10000)

            WebRTCFPS: 60, // Максимальный FPS 30 для стабильности (default: 60)

            // ============================================
            // НАСТРОЙКИ КАЧЕСТВА ВИДЕО
            // ============================================
            VideoScalingFactor: 1.0, // Масштаб видео (1.0 = 100%, 0.75 = 75%) (default: 1.0)

            // Настройки кодирования
            KeyframeInterval: 500, // Интервал ключевых кадров в мс (default: 2000)
            // Меньше значение = чаще I-frames = стабильнее картинка, но больше трафик

            // ============================================
            // НАСТРОЙКИ УПРАВЛЕНИЯ МЫШЬЮ
            // ============================================
            SuppressBrowserKeys: true, // Блокировать браузерные горячие клавиши (default: true)
            IsQualityController: false, // Управление качеством на клиенте (default: false)

            // ============================================
            // НАСТРОЙКИ РАЗРЕШЕНИЯ И VIEWPORT
            // ============================================
            MatchViewportResolution: true, // Подстраивать под разрешение viewport (default: false)
            // Критично для планшетов с разными разрешениями

            // ============================================
            // НАСТРОЙКИ АВТОПОДКЛЮЧЕНИЯ СТРИМЕРА
            // ============================================
            StreamerAutoJoin: true, // Автоматически подключаться к стримеру (default: false)

            // ============================================
            // НАСТРОЙКИ XR (VR/AR)
            // ============================================
            XRControllerInput: false, // Включить поддержку XR контроллеров (default: false)

            // ============================================
            // ЭКСПЕРИМЕНТАЛЬНЫЕ НАСТРОЙКИ
            // ============================================
            TimeoutIfIdle: false, // Таймаут при бездействии (default: false)
            MaxReconnectAttempts: 3, // Максимум попыток переподключения (default: 3)

            // Настройки задержки
            MaxLatency: 200, // Максимальная задержка в мс (default: не задан)
            MinLatency: 0, // Минимальная задержка в мс (default: 0)

            // ============================================
            // НАСТРОЙКИ ДЛЯ TOUCH УСТРОЙСТВ
            // ============================================
            TouchScreenInput: true, // Включить поддержку тач-скрина (default: true)
            GamepadInput: false, // Включить поддержку геймпада (default: true)

            // ============================================
            // ДОПОЛНИТЕЛЬНЫЕ НАСТРОЙКИ
            // ============================================
            AFKTimeout: 0, // Таймаут AFK в секундах (0 = выключен) (default: 0)
            WaitForStreamer: true, // Ждать подключения стримера (default: true)

            // Статистика и мониторинг
            PrintStats: false, // Выводить статистику в консоль (default: false)
            ShowTextOverlays: false, // Показывать текстовые оверлеи (default: false)

            // ============================================
            // ПРИМЕЧАНИЯ ПО ОПТИМИЗАЦИИ
            // ============================================
            // Для телефонов:
            //   - MinBitrate: 300000, MaxBitrate: 5000000
            //   - VideoScalingFactor: 0.75, WebRTCFPS: 30
            //   - KeyframeInterval: 1500

            // Для планшетов (текущие):
            //   - MinBitrate: 500000, MaxBitrate: 8000000
            //   - VideoScalingFactor: 1.0, WebRTCFPS: 30
            //   - KeyframeInterval: 1000

            // Для десктопов:
            //   - MinBitrate: 1000000, MaxBitrate: 20000000
            //   - VideoScalingFactor: 1.0, WebRTCFPS: 60
            //   - KeyframeInterval: 2000

            // При проблемах с картинкой:
            //   - Уменьшите KeyframeInterval до 500
            //   - Попробуйте PreferVP8: true вместо H264
            //   - Снизьте WebRTCFPS до 24
         },
      });

      pixelStreaming = new PixelStreaming(config, {
         videoElementParent: videoContainer.value,
      });

      // Прямая настройка WebRTC для планшетов (расширенная конфигурация)
      if (
         pixelStreaming.config &&
         pixelStreaming.config.setOptionSettingValue
      ) {
         // Настройка параметров кодировщика для стабильной передачи
         try {
            // Принудительная отправка ключевых кадров
            pixelStreaming.config.setOptionSettingValue(
               "WebRTC",
               "DegradationPreference",
               "maintain-framerate"
            );
            pixelStreaming.config.setOptionSettingValue("WebRTC", "MaxFPS", 60);
            pixelStreaming.config.setOptionSettingValue("WebRTC", "MinQP", 1);
            pixelStreaming.config.setOptionSettingValue("WebRTC", "MaxQP", 100);
         } catch (e) {
            console.warn(
               "Не удалось применить расширенные настройки WebRTC:",
               e
            );
         }
      }

      pixelStreaming.addEventListener("webRtcConnected", () => {
         isConnected.value = true;
         isConnecting.value = false;
         // Настраиваем перехват координат мыши после подключения
         setTimeout(() => setupMouseInterception(), 100);

         // Автоматически включаем fullscreen при подключении (если включено в настройках)
         setTimeout(() => {
            const savedSettings = localStorage.getItem("goodiniSettings");
            const fullscreenEnabled = savedSettings
               ? JSON.parse(savedSettings).fullscreenEnabled !== false
               : true; // По умолчанию включен

            if (fullscreenEnabled) {
               const elem = document.documentElement;
               if (elem.requestFullscreen) {
                  elem.requestFullscreen().catch(() => {});
               } else if (elem.webkitRequestFullscreen) {
                  elem.webkitRequestFullscreen();
               }
            }
         }, 500);
      });

      pixelStreaming.addEventListener("webRtcDisconnected", () => {
         isConnected.value = false;
         isConnecting.value = false;
      });

      pixelStreaming.addEventListener("playStreamError", () => {
         errorMessage.value = "Ошибка воспроизведения стрима";
         isConnecting.value = false;
      });

      pixelStreaming.addEventListener("dataChannelMessage", (data) => {
         lastMessage.value =
            typeof data === "string" ? data : JSON.stringify(data);

         const time = new Date().toLocaleTimeString();
         receivedMessages.value.unshift({
            time: time,
            data:
               typeof data === "string" ? data : JSON.stringify(data, null, 2),
         });

         if (receivedMessages.value.length > 10) {
            receivedMessages.value.pop();
         }
      });

      pixelStreaming.addResponseEventListener("response", (response) => {
         lastMessage.value =
            typeof response === "string" ? response : JSON.stringify(response);

         const time = new Date().toLocaleTimeString();
         receivedMessages.value.unshift({
            time: time,
            data:
               typeof response === "string"
                  ? response
                  : JSON.stringify(response, null, 2),
         });

         if (receivedMessages.value.length > 10) {
            receivedMessages.value.pop();
         }
      });

      // Мониторинг статистики WebRTC для диагностики
      pixelStreaming.addEventListener("statsReceived", (stats) => {
         // Логирование статистики для отладки на планшетах
         if (stats && stats.inboundVideoStats) {
            const videoStats = stats.inboundVideoStats;
            if (
               videoStats.framesReceived === 0 ||
               videoStats.framesDecoded === 0
            ) {
               console.warn("⚠️ Планшет не получает кадры:", videoStats);
            }
         }
      });

      pixelStreaming.connect();
   } catch (error) {
      errorMessage.value = `Ошибка: ${error.message}`;
      isConnecting.value = false;
   }
};

const sendJsonData = () => {
   if (pixelStreaming) {
      pixelStreaming.emitUIInteraction({ action: "jump" });
   }
};

const sendToEngine = (data) => {
   console.log("🎮 sendToEngine вызван, данные:", data);
   if (pixelStreaming) {
      console.log("✅ Отправляем через pixelStreaming.emitUIInteraction");
      pixelStreaming.emitUIInteraction(data);
   } else {
      console.error("❌ pixelStreaming не инициализирован!");
   }
};

onBeforeUnmount(() => {
   if (pixelStreaming) {
      pixelStreaming.disconnect();
   }
});
</script>

<style scoped>
.pixel-streaming-container {
   width: 100%;
   height: 100vh;
   display: flex;
   align-items: center;
   justify-content: center;
   background: #1a1a1a;
}

.video-wrapper {
   position: relative;
   width: 100%;
   height: 100%;
   display: flex;
   align-items: center;
   justify-content: center;
}

.video-container {
   width: 100%;
   height: 100%;
   position: absolute;
   top: 0;
   left: 0;
   z-index: 0; /* КРИТИЧНО: Video должен быть под UI элементами */

   /* Изоляция слоя для планшетов - предотвращает блокировку рендеринга overlay */
   isolation: isolate;
}

.video-container :deep(video) {
   /* Базовый transform без избыточных оптимизаций */
   transform: scaleX(-1);

   /* Явные размеры для планшетов */
   width: 100%;
   height: 100%;
   object-fit: contain;
   display: block; /* Важно для правильного рендеринга */

   /* КРИТИЧНО для планшетов: явное указание что это video layer */
   position: relative;
   z-index: 1;
}

.overlay {
   position: absolute;
   top: 0;
   left: 0;
   width: 100%;
   height: 100%;
   display: flex;
   align-items: center;
   justify-content: center;
   background: rgba(0, 0, 0, 0.8);
   z-index: 10;
}

.connection-panel {
   background: #2a2a2a;
   padding: 2rem;
   border-radius: 8px;
   box-shadow: 0 4px 6px rgba(0, 0, 0, 0.3);
   min-width: 400px;
}

.connection-panel h2 {
   margin: 0 0 1.5rem 0;
   color: #fff;
   text-align: center;
}

.input-group {
   margin-bottom: 1.5rem;
}

.input-group label {
   display: block;
   color: #ccc;
   margin-bottom: 0.5rem;
   font-size: 0.9rem;
}

.input-group input {
   width: 100%;
   padding: 0.75rem;
   border: 1px solid #444;
   border-radius: 4px;
   background: #1a1a1a;
   color: #fff;
   font-size: 1rem;
   box-sizing: border-box;
}

.input-group input:focus {
   outline: none;
   border-color: #646cff;
}

.connect-btn {
   width: 100%;
   padding: 0.75rem;
   background: #646cff;
   color: white;
   border: none;
   border-radius: 4px;
   font-size: 1rem;
   cursor: pointer;
   transition: background 0.3s;
   user-select: none;
}

.connect-btn:hover:not(:disabled) {
   background: #535bf2;
}

.connect-btn:disabled {
   background: #444;
   cursor: not-allowed;
}

.error-message {
   margin-top: 1rem;
   padding: 0.75rem;
   background: #ff4444;
   color: white;
   border-radius: 4px;
   text-align: center;
}

.video-container :deep(video) {
   transform: scaleX(-1);

   /* Оптимизации для планшетов */
   will-change: transform;
   backface-visibility: hidden;
   -webkit-backface-visibility: hidden;
   transform-style: preserve-3d;
   -webkit-transform-style: preserve-3d;

   /* Принудительное использование аппаратного ускорения */
   -webkit-transform: translateZ(0) scaleX(-1);
   -moz-transform: translateZ(0) scaleX(-1);

   /* Предотвращение мерцания на планшетах */
   -webkit-font-smoothing: antialiased;
   -moz-osx-font-smoothing: grayscale;
}

.overlay {
   position: absolute;

   top: 0;

   left: 0;

   width: 100%;

   height: 100%;

   display: flex;

   align-items: center;

   justify-content: center;

   background: rgba(0, 0, 0, 0.8);

   z-index: 10;
}

.connection-panel {
   background: #2a2a2a;

   padding: 2rem;

   border-radius: 8px;

   box-shadow: 0 4px 6px rgba(0, 0, 0, 0.3);

   min-width: 400px;
}

.connection-panel h2 {
   margin: 0 0 1.5rem 0;

   color: #fff;

   text-align: center;
}

.input-group {
   margin-bottom: 1.5rem;
}

.input-group label {
   display: block;

   color: #ccc;

   margin-bottom: 0.5rem;

   font-size: 0.9rem;
}

.input-group input {
   width: 100%;

   padding: 0.75rem;

   border: 1px solid #444;

   border-radius: 4px;

   background: #1a1a1a;

   color: #fff;

   font-size: 1rem;

   box-sizing: border-box;
}

.input-group input:focus {
   outline: none;

   border-color: #646cff;
}

.connect-btn {
   width: 100%;

   padding: 0.75rem;

   background: #646cff;

   color: white;

   border: none;

   border-radius: 4px;

   font-size: 1rem;

   cursor: pointer;

   transition: background 0.3s;

   user-select: none;
}

.connect-btn:hover:not(:disabled) {
   background: #535bf2;
}

.connect-btn:disabled {
   background: #444;

   cursor: not-allowed;
}

.error-message {
   margin-top: 1rem;

   padding: 0.75rem;

   background: #ff4444;

   color: white;

   border-radius: 4px;

   text-align: center;
}
</style>
