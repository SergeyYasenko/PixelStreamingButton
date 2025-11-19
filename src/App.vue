<template>
   <div class="app">
      <div
         class="connection-status"
         :class="{ connected: isConnected, disconnected: !isConnected }"
      >
         <span class="status-dot"></span>
         <span class="status-text">
            {{ isConnected ? "Подключено" : "Не подключено" }}
         </span>
      </div>
      <div class="button-container">
         <button
            v-for="(button, index) in buttons"
            :key="index"
            class="stream-button"
            @click="sendMessage(button.value)"
            :disabled="!isConnected"
         >
            {{ button.label }}
         </button>
      </div>
   </div>
</template>

<script setup>
import { onMounted, onUnmounted, ref } from "vue";

// Список кнопок с названиями
const buttons = ref([
   { label: "Основная пролетка", value: "1" },
   { label: "Фок", value: "2" },
   { label: "Стадион", value: "3" },
   { label: "Йога", value: "4" },
   { label: "Теннис", value: "5" },
   { label: "Шабада", value: "6" },
   { label: "Благоустройство", value: "7" },
]);

// WebSocket соединение для PixelStreaming
const ws = ref(null);
const isConnected = ref(false);
const wsUrl = ref("ws://localhost:8888"); // URL вашего PixelStreaming сервера

onMounted(() => {
   // Автоматическое подключение при монтировании компонента
   connectToPixelStreaming();
});

onUnmounted(() => {
   disconnectFromPixelStreaming();
});

const connectToPixelStreaming = () => {
   try {
      if (ws.value && ws.value.readyState === WebSocket.OPEN) {
         console.log("Уже подключено к PixelStreaming");
         return;
      }

      console.log(`Подключение к PixelStreaming: ${wsUrl.value}`);
      ws.value = new WebSocket(wsUrl.value);

      ws.value.onopen = () => {
         isConnected.value = true;
         console.log("✓ Подключено к PixelStreaming");
         console.log("WebSocket URL:", wsUrl.value);
      };

      ws.value.onclose = () => {
         isConnected.value = false;
         console.log("✗ Отключено от PixelStreaming");
      };

      ws.value.onerror = (error) => {
         console.error("Ошибка WebSocket:", error);
         console.error(
            "Проверьте, что PixelStreaming сервер запущен на:",
            wsUrl.value
         );
         isConnected.value = false;
         alert(
            `Ошибка подключения к PixelStreaming серверу.\nURL: ${wsUrl.value}\n\nУбедитесь, что сервер запущен и доступен.`
         );
      };

      ws.value.onmessage = (event) => {
         console.log("Получено сообщение от сервера:", event.data);
      };
   } catch (error) {
      console.error("Ошибка подключения к PixelStreaming:", error);
      isConnected.value = false;
   }
};

const disconnectFromPixelStreaming = () => {
   if (ws.value) {
      try {
         ws.value.close();
         ws.value = null;
         isConnected.value = false;
         console.log("Отключено от PixelStreaming");
      } catch (error) {
         console.error("Ошибка при отключении:", error);
      }
   }
};

const sendMessage = (message) => {
   console.log(`Попытка отправить сообщение: ${message}`);
   console.log(`Состояние WebSocket:`, ws.value ? ws.value.readyState : "null");
   console.log(`isConnected:`, isConnected.value);

   try {
      if (ws.value && ws.value.readyState === WebSocket.OPEN) {
         // PixelStreaming может принимать сообщения в разных форматах
         // Попробуем несколько вариантов

         // Вариант 1: JSON с ConsoleCommand
         const payloadJson = JSON.stringify({
            ConsoleCommand: message,
         });

         // Вариант 2: Просто строка (для некоторых версий PixelStreaming)
         const payloadString = message;

         // Отправляем JSON формат (стандартный для PixelStreaming)
         ws.value.send(payloadJson);
         console.log(`✓ Отправлено сообщение (JSON):`, payloadJson);
         console.log(`✓ Отправлено сообщение (строка):`, message);
      } else {
         // Если не подключено, пытаемся подключиться
         console.warn(
            `WebSocket не подключен. Состояние:`,
            ws.value ? ws.value.readyState : "не инициализирован"
         );

         if (!ws.value || ws.value.readyState === WebSocket.CLOSED) {
            console.log("Попытка подключения...");
            connectToPixelStreaming();

            // Ждем подключения и пытаемся отправить снова
            const maxAttempts = 10;
            let attempts = 0;

            const trySend = setInterval(() => {
               attempts++;

               if (ws.value && ws.value.readyState === WebSocket.OPEN) {
                  clearInterval(trySend);
                  const payloadJson = JSON.stringify({
                     ConsoleCommand: message,
                  });
                  ws.value.send(payloadJson);
                  console.log(
                     `✓ Отправлено сообщение (после подключения): ${message}`
                  );
               } else if (attempts >= maxAttempts) {
                  clearInterval(trySend);
                  console.error(
                     `Не удалось подключиться за ${maxAttempts} попыток. Сообщение: ${message}`
                  );
                  alert(
                     `Не удалось подключиться к PixelStreaming серверу.\nПроверьте, что сервер запущен на ${wsUrl.value}`
                  );
               }
            }, 200);
         } else if (ws.value.readyState === WebSocket.CONNECTING) {
            console.log("WebSocket подключается, ожидание...");
            // Ждем завершения подключения
            ws.value.addEventListener(
               "open",
               () => {
                  const payloadJson = JSON.stringify({
                     ConsoleCommand: message,
                  });
                  ws.value.send(payloadJson);
                  console.log(
                     `✓ Отправлено сообщение (после ожидания подключения): ${message}`
                  );
               },
               { once: true }
            );
         } else {
            console.error(
               `WebSocket в неожиданном состоянии:`,
               ws.value.readyState
            );
            alert(
               `Ошибка подключения к PixelStreaming.\nСостояние: ${ws.value.readyState}\nПроверьте URL: ${wsUrl.value}`
            );
         }
      }
   } catch (error) {
      console.error("Ошибка отправки сообщения:", error);
      alert(`Ошибка отправки: ${error.message}`);
   }
};
</script>

<style>
* {
   margin: 0;
   padding: 0;
   box-sizing: border-box;
}

body {
   font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Oxygen,
      Ubuntu, Cantarell, sans-serif;
   background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
   min-height: 100vh;
   display: flex;
   align-items: center;
   justify-content: center;
}

.app {
   width: 100%;
   height: 100vh;
   display: flex;
   flex-direction: column;
   align-items: center;
   justify-content: center;
   position: relative;
}

.connection-status {
   position: absolute;
   top: 20px;
   right: 20px;
   display: flex;
   align-items: center;
   gap: 8px;
   padding: 8px 16px;
   background: rgba(0, 0, 0, 0.5);
   border-radius: 20px;
   color: white;
   font-size: 0.9rem;
   z-index: 100;
}

.status-dot {
   width: 10px;
   height: 10px;
   border-radius: 50%;
   background: #ff4444;
   animation: pulse 2s infinite;
}

.connection-status.connected .status-dot {
   background: #44ff44;
   box-shadow: 0 0 10px rgba(68, 255, 68, 0.5);
}

.connection-status.disconnected .status-dot {
   background: #ff4444;
   box-shadow: 0 0 10px rgba(255, 68, 68, 0.5);
}

@keyframes pulse {
   0%,
   100% {
      opacity: 1;
   }
   50% {
      opacity: 0.5;
   }
}

.button-container {
   display: grid;
   grid-template-columns: repeat(3, 1fr);
   gap: 20px;
   max-width: 1200px;
   width: 100%;
   padding: 20px;
}

.stream-button {
   min-height: 120px;
   min-width: 120px;
   width: 100%;
   height: auto;
   font-size: 1.3rem;
   font-weight: bold;
   color: white;
   background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
   border: none;
   border-radius: 20px;
   cursor: pointer;
   transition: all 0.3s ease;
   box-shadow: 0 10px 30px rgba(0, 0, 0, 0.3);
   position: relative;
   overflow: hidden;
   padding: 20px 15px;
   display: flex;
   align-items: center;
   justify-content: center;
   text-align: center;
   line-height: 1.3;
   word-wrap: break-word;
   hyphens: auto;
}

.stream-button::before {
   content: "";
   position: absolute;
   top: 50%;
   left: 50%;
   width: 0;
   height: 0;
   border-radius: 50%;
   background: rgba(255, 255, 255, 0.3);
   transform: translate(-50%, -50%);
   transition: width 0.6s, height 0.6s;
}

.stream-button:hover {
   transform: translateY(-5px) scale(1.05);
   box-shadow: 0 15px 40px rgba(0, 0, 0, 0.4);
}

/* Отключаем hover эффекты на touch устройствах */
@media (hover: none) {
   .stream-button:hover {
      transform: none;
   }
}

.stream-button:active {
   transform: translateY(-2px) scale(1.02);
}

.stream-button:active::before {
   width: 300px;
   height: 300px;
}

.stream-button:disabled {
   opacity: 0.6;
   cursor: not-allowed;
   transform: none !important;
}

.stream-button:nth-child(1) {
   background: linear-gradient(135deg, #f093fb 0%, #f5576c 100%);
}

.stream-button:nth-child(2) {
   background: linear-gradient(135deg, #4facfe 0%, #00f2fe 100%);
}

.stream-button:nth-child(3) {
   background: linear-gradient(135deg, #43e97b 0%, #38f9d7 100%);
}

.stream-button:nth-child(4) {
   background: linear-gradient(135deg, #fa709a 0%, #fee140 100%);
}

.stream-button:nth-child(5) {
   background: linear-gradient(135deg, #30cfd0 0%, #330867 100%);
}

.stream-button:nth-child(6) {
   background: linear-gradient(135deg, #a8edea 0%, #fed6e3 100%);
   color: #333;
}

.stream-button:nth-child(7) {
   background: linear-gradient(135deg, #ff9a9e 0%, #fecfef 100%);
   color: #333;
}

/* Большие планшеты и маленькие десктопы (1024px - 1200px) */
@media (max-width: 1200px) {
   .button-container {
      max-width: 1000px;
      gap: 18px;
      padding: 18px;
   }

   .stream-button {
      min-height: 110px;
      min-width: 110px;
      font-size: 1.2rem;
      border-radius: 18px;
      padding: 18px 14px;
   }
}

/* Планшеты в портретной ориентации (768px - 1024px) */
@media (max-width: 1024px) {
   .button-container {
      grid-template-columns: repeat(3, 1fr);
      max-width: 900px;
      gap: 16px;
      padding: 16px;
   }

   .stream-button {
      min-height: 100px;
      min-width: 100px;
      font-size: 1.15rem;
      border-radius: 16px;
      padding: 16px 12px;
   }
}

/* Планшеты в альбомной ориентации (768px - 1024px, landscape) */
@media (max-width: 1024px) and (orientation: landscape) {
   .button-container {
      grid-template-columns: repeat(4, 1fr);
      max-width: 100%;
      gap: 14px;
      padding: 14px;
   }

   .stream-button {
      min-height: 90px;
      min-width: 90px;
      font-size: 1rem;
      padding: 14px 10px;
   }
}

/* Планшеты в альбомной ориентации и большие телефоны (600px - 768px) */
@media (max-width: 768px) {
   .button-container {
      grid-template-columns: repeat(3, 1fr);
      gap: 14px;
      padding: 14px;
   }

   .stream-button {
      min-height: 90px;
      min-width: 90px;
      font-size: 1rem;
      border-radius: 14px;
      padding: 14px 10px;
   }
}

/* Мобильные устройства (до 600px) */
@media (max-width: 600px) {
   .button-container {
      grid-template-columns: repeat(2, 1fr);
      gap: 12px;
      padding: 12px;
   }

   .stream-button {
      min-height: 80px;
      min-width: 80px;
      font-size: 0.95rem;
      border-radius: 12px;
      padding: 12px 8px;
   }
}

/* Маленькие мобильные устройства (до 480px) */
@media (max-width: 480px) {
   .button-container {
      grid-template-columns: repeat(2, 1fr);
      gap: 10px;
      padding: 10px;
   }

   .stream-button {
      min-height: 70px;
      min-width: 70px;
      font-size: 0.85rem;
      border-radius: 10px;
      padding: 10px 6px;
      line-height: 1.1;
   }
}

/* Очень маленькие экраны (до 360px) */
@media (max-width: 360px) {
   .button-container {
      gap: 8px;
      padding: 8px;
   }

   .stream-button {
      min-height: 60px;
      min-width: 60px;
      font-size: 0.75rem;
      border-radius: 8px;
      padding: 8px 5px;
   }
}

/* Адаптивные стили для индикатора статуса */
@media (max-width: 768px) {
   .connection-status {
      top: 10px;
      right: 10px;
      font-size: 0.8rem;
      padding: 6px 12px;
   }

   .status-dot {
      width: 8px;
      height: 8px;
   }
}

@media (max-width: 480px) {
   .connection-status {
      top: 5px;
      right: 5px;
      font-size: 0.75rem;
      padding: 5px 10px;
   }
}
</style>
