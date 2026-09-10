# Общий блокнот (GitHub Pages)

Одностраничный сайт: **один человек пишет текст — все, кто открыл страницу, видят его сразу**, без перезагрузки.

GitHub Pages отдаёт только статические файлы, поэтому «общее» состояние хранится снаружи —
в публичном MQTT-брокере (HiveMQ) через WebSocket. Брокер запоминает последнее сообщение
(*retained*), поэтому новый посетитель получает текущий текст сразу при заходе.
Своего сервера, регистрации и оплаты не требуется.

## Как это работает

| Роль | Ссылка | Что делает |
|---|---|---|
| Редактор (ваш ПК) | `https://dimidrol2252.github.io/#admin` | видит поле ввода, публикует текст |
| Читатели | `https://dimidrol2252.github.io/` | видят текст живьём, редактировать не могут |

Текст публикуется автоматически через ~0.8 с после того, как вы перестали печатать
(или сразу по кнопке «Опубликовать»).

## Публикация на GitHub

```bash
cd notepad_share
git init
git add .
git commit -m "Общий блокнот"
git branch -M main
git remote add origin https://github.com/Dimidrol2252/Dimidrol2252.github.io.git
git push -u origin main
```

Дальше в репозитории: **Settings → Pages → Source: Deploy from a branch → Branch: `main` / `/ (root)` → Save**.
Через минуту сайт будет доступен по адресу `https://dimidrol2252.github.io/`.

## Обязательно поменяйте `room`

В `index.html`, вверху скрипта:

```js
const CONFIG = {
  broker: "wss://broker.hivemq.com:8884/mqtt",
  room: "73beefa132ff2d64"   // ваш личный канал
};
```

Сгенерировать свою:

```bash
openssl rand -hex 8
```

`room` — это ваш канал. Пока строка случайная, посторонний в неё не попадёт.

## Что важно понимать про ограничения

- **Брокер публичный и без авторизации.** Кто откроет исходный код страницы (а он открыт всегда,
  это статический сайт), увидит `room` и технически сможет писать в тот же канал.
  Для «доски объявлений» это нормально, для чего-то ответственного — см. вариант с Firebase ниже.
- **Не публикуйте пароли и личные данные** — трафик идёт через чужой бесплатный брокер.
- Retained-сообщение живёт на брокере, но публичный HiveMQ ничего не гарантирует:
  после долгого простоя или перезапуска брокера текст может пропасть — достаточно опубликовать заново.
- Размер текста лучше держать в пределах нескольких сотен килобайт.

## Вариант понадёжнее: Firebase Realtime Database

Если нужны настоящие права доступа (писать может только вы) и гарантированное хранение —
бесплатного тарифа Firebase хватает с запасом.

1. Создайте проект на <https://console.firebase.google.com>, добавьте Realtime Database.
2. Правила базы — читать всем, писать только авторизованному:

```json
{
  "rules": {
    "note": {
      ".read": true,
      ".write": "auth != null && auth.uid === 'ВАШ_UID'"
    }
  }
}
```

3. В `index.html` замените блок с `mqtt.connect(...)` на:

```html
<script type="module">
import { initializeApp } from "https://www.gstatic.com/firebasejs/10.12.2/firebase-app.js";
import { getDatabase, ref, onValue, set } from "https://www.gstatic.com/firebasejs/10.12.2/firebase-database.js";

const app = initializeApp({ /* конфиг из консоли Firebase */ });
const noteRef = ref(getDatabase(app), "note");

onValue(noteRef, (snap) => {
  const data = snap.val() || { text: "", ts: 0 };
  /* здесь вызвать render(data.text, data.ts) или подставить в textarea */
});

// публикация:
// set(noteRef, { text: input.value, ts: Date.now() });
</script>
```

Остальная разметка и стили не меняются — меняется только транспорт.

## Локальная проверка

```bash
python3 -m http.server 8000
```

Откройте `http://localhost:8000/#admin` в одном окне и `http://localhost:8000/` в другом.
