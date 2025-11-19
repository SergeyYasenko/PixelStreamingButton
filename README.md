# PixelStreaming Button App

Vue.js приложение с 6 кнопками для отправки сообщений через PixelStreaming.

## Установка

```bash
npm install
```

## Запуск

```bash
npm run dev
```

Приложение откроется на `http://localhost:3000`

## Настройка PixelStreaming

1. Откройте файл `src/App.vue`
2. Найдите переменную `wsUrl` в секции `<script setup>`
3. Измените URL на адрес вашего PixelStreaming сервера:

```javascript
const wsUrl = ref("ws://your-server:8888");
```

4. Для автоматического подключения при загрузке страницы, раскомментируйте строку в `onMounted`:

```javascript
onMounted(() => {
   connectToPixelStreaming(); // Раскомментируйте эту строку
});
```

## Использование

- На странице отображаются 6 кнопок с номерами от 1 до 6
- При клике на кнопку отправляется соответствующая строка ("1", "2", "3", "4", "5", "6") через WebSocket в формате JSON:
  ```json
  {
    "ConsoleCommand": "1"
  }
  ```
- Если PixelStreaming не подключен, приложение попытается подключиться автоматически
- Если подключение не удалось, сообщение будет показано в alert для тестирования

## Структура проекта

```
PixelStreamingButton/
├── index.html          # Точка входа HTML
├── package.json        # Зависимости проекта
├── vite.config.js     # Конфигурация Vite
├── .gitignore         # Игнорируемые файлы
├── README.md          # Документация
└── src/
    ├── main.js        # Инициализация Vue приложения
    └── App.vue        # Главный компонент с кнопками
```

## Сборка для продакшена

```bash
npm run build
```

Собранные файлы будут в папке `dist`

