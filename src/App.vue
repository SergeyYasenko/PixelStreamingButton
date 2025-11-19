<template>
   <div class="app">
      <div class="button-container">
         <button
            v-for="(button, index) in buttons"
            :key="index"
            class="stream-button"
            @click="sendMessage(button.value)"
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
   // Раскомментируйте следующую строку для автоматического подключения
   // connectToPixelStreaming();
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
      };

      ws.value.onclose = () => {
         isConnected.value = false;
         console.log("✗ Отключено от PixelStreaming");
      };

      ws.value.onerror = (error) => {
         console.error("Ошибка WebSocket:", error);
         isConnected.value = false;
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
   try {
      if (ws.value && ws.value.readyState === WebSocket.OPEN) {
         // Отправляем сообщение в формате, который ожидает PixelStreaming
         // Обычно это JSON с полем ConsoleCommand или просто строка
         const payload = JSON.stringify({
            ConsoleCommand: message,
         });
         ws.value.send(payload);
         console.log(`✓ Отправлено сообщение: ${message}`);
      } else {
         // Если не подключено, пытаемся подключиться
         if (!ws.value || ws.value.readyState === WebSocket.CLOSED) {
            connectToPixelStreaming();
            // Ждем немного и пытаемся отправить снова
            setTimeout(() => {
               if (ws.value && ws.value.readyState === WebSocket.OPEN) {
                  const payload = JSON.stringify({
                     ConsoleCommand: message,
                  });
                  ws.value.send(payload);
                  console.log(
                     `✓ Отправлено сообщение (после подключения): ${message}`
                  );
               } else {
                  console.warn(
                     `PixelStreaming не подключен. Сообщение (для теста): ${message}`
                  );
               }
            }, 500);
         } else {
            console.warn(
               `PixelStreaming не подключен. Сообщение (для теста): ${message}`
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
   align-items: center;
   justify-content: center;
}

.button-container {
   display: grid;
   grid-template-columns: repeat(3, 1fr);
   gap: 20px;
   max-width: 800px;
   width: 100%;
   padding: 20px;
}

.stream-button {
   aspect-ratio: 1;
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
   padding: 10px;
   display: flex;
   align-items: center;
   justify-content: center;
   text-align: center;
   line-height: 1.2;
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

.stream-button:active {
   transform: translateY(-2px) scale(1.02);
}

.stream-button:active::before {
   width: 300px;
   height: 300px;
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

@media (max-width: 768px) {
   .button-container {
      grid-template-columns: repeat(2, 1fr);
      gap: 15px;
      padding: 15px;
   }

   .stream-button {
      font-size: 1.1rem;
   }
}
</style>
