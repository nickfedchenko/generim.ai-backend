# Инструкция по интеграции нового сервиса генерации Generim.ai с RabbitMQ

В данной инструкции описывается порядок создания необходимых очередей и биндингов в RabbitMQ, а также правила обмена сообщениями между сервисом и системой генерации.

## 1. Создание очередей

Для каждого нового сервиса генерации необходимо создать четыре очереди — две для production и две для staging.
Формат имен очередей:

* **Production**

  * `creagen.task.servicename`
  * `creagen.task.servicename.result`
* **Staging**

  * `staging.creagen.task.servicename`
  * `staging.creagen.task.servicename.result`

Где `servicename` — уникальное имя вашего микросервиса.

## 2. Настройка биндингов

Все очереди должны быть привязаны к exchange `amq.direct`.
Для каждой очереди создается биндинг с routing key, полностью совпадающим с ее названием.

Пример:


| Queue name                                | Exchange     | Routing key                               |
| ----------------------------------------- | ------------ | ----------------------------------------- |
| `creagen.task.servicename`                | `amq.direct` | `creagen.task.servicename`                |
| `creagen.task.servicename.result`         | `amq.direct` | `creagen.task.servicename.result`         |
| `staging.creagen.task.servicename`        | `amq.direct` | `staging.creagen.task.servicename`        |
| `staging.creagen.task.servicename.result` | `amq.direct` | `staging.creagen.task.servicename.result` |

## 3. Формат обмена сообщениями

### 3.1. Очередь `creagen.task.servicename`

В эту очередь система Generim.ai отправляет задачи на генерацию.
Сообщение представляет собой JSON с параметрами, необходимыми вашему сервису.

**Пример сообщения:**

```json
 {
  "id": 12221,
  "queueId": "019ac91e-c6a1-7391-ae58-2252d80ba1ae", // ID группы задач на генерацию
  "taskId": "019ac91e-c6a6-719a-9ea6-536c8682b586", // ID задачи на генерацию
  "status": "new",
  "tokens": 1.772,
  "error": null,
  "startedAt": "2025-11-28T06:20:28+00:00",
  "finishedAt": null,
  "duration": null,
  "model": "seedream_40_edit", // ID ИИ-модели
  "isFavorite": false,
  "inputPayload": { // сырые данные
    "prompt": "make a photo of the man driving the car down the california coastline",
    "userId": 52,
    "imageSize": "auto_2K",
    "imageUrls": [
      "https://storage.googleapis.com/falserverless/example_inputs/nano-banana-edit-input.png",
      "https://storage.googleapis.com/falserverless/example_inputs/nano-banana-edit-input-2.png"
    ],
    "sessionId": 4,
    "customSize": null,
    "resolution": "2",
    "aspectRatio": "Авто",
    "numberOfResults": 2,
    "translatedPrompt": "make a photo of the man driving the car down the california coastline"
  },
  "payload": { // подготовленные данные
    "seed": 3926907256,
    "prompt": "make a photo of the man driving the car down the california coastline",
    "image_size": "auto_2K",
    "image_urls": [
      "https://storage.googleapis.com/falserverless/example_inputs/nano-banana-edit-input.png",
      "https://storage.googleapis.com/falserverless/example_inputs/nano-banana-edit-input-2.png"
    ],
    "max_images": 1,
    "num_images": 1,
    "enable_safety_checker": true
  },
  "videoPreview": null,
  "results": [],
  "maskUrl": null,
  "provider": "fal", // сервис генерации
  "falCode": "fal-ai/bytedance/seedream/v4/edit",
  "resultQueue": "staging.creagen.task.seedream.result" // наименование очереди для отправки результата
}
```

### 3.2. Очередь `creagen.task.servicename.result`

Ваш сервис обязан отправлять результат выполнения задачи в эту очередь. Результат отправляется при любом смене статуса (генерация, завершено, ошибка).
Формат сообщения — JSON с идентификатором задачи и результатом генерации.

**Пример сообщения:**

```json
{
  "id": 12222,
  "eventId": "task.12222", // ID события
  "status": "completed", // статус генерации
  "error": null, // текст ошибки
  "taskId": "019ac927-d775-70b7-86f0-689216f19b45", // ID задачи на генерацию
  "queueId": "019ac927-d773-73a9-bc76-e7419eac79a0", // ID группы задач на генерацию
  "results": [ // массив результатов
    {
      "url": "https://v3b.fal.media/files/b/koala/LqTpw4NJaw_JPLNki7ZzK_output.mp4",
      "type": "video" // тип результата (image, video)
    }
  ],
  "falRequestId": null,
  "duration": 59, // длительность генерации в секундах
  "megapixels": null, // количество мегапикселей (изображения, видео)
  "timingInference": null, // время потраченное на генерацию (если тарификация за секунду генерации)
  "resultQueue": "staging.creagen.task.kling_fal.result"
}
```

## 4. Статусы генераций


| Статус | Описание                                     |
| ------------ | ---------------------------------------------------- |
| `new`        | Новая задача                              |
| `processing` | Генерация                                   |
| `completed`  | Генерация успешно завершена |
| `failed`     | Ошибка генерации                      |

## 5. Общие рекомендации

* Все сообщения должны быть сериализованы в JSON.
* taskId обязателен и должен совпадать в запросе и ответе.
* Ошибки генерации рекомендуется отправлять в ту же result-очередь с status: "error" и описанием проблемы.
* Сервис должен автоматически перезапускаться в случае аварийного завершения работы или перезагрузки сервера.
* Сервис не должен удалять или отменять задачу в случае, если результат не был доставлен получателю.
* Если сервис сохраняет файлы, то обеспечить их автоматическое удаление через заданный интервал времени. Бэкенд Generim.ai сохраняет результаты в своем хранилище и не использует хранилище сервиса.
