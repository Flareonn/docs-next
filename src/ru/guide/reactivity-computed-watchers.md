# Вычисляемые свойства и методы-наблюдатели

> Этот раздел использует синтаксис [однофайловых компонентов](single-file-component.md) для примеров кода

## Вычисляемые значения

Иногда требуется состояние, зависящее от другого состояния — во Vue это реализуется с помощью [свойства computed](computed.md#вычисляемые-своиства) компонента. Для создания вычисляемого значения напрямую можно использовать метод `computed`: он получает функцию геттер и возвращает реактивный иммутабельный [ref](reactivity-fundamentals.md#создание-автономных-ссылок-на-реактивные-значения) объект для возвращаемого значения из геттера.

```js
const count = ref(1)
const plusOne = computed(() => count.value + 1)

console.log(plusOne.value) // 2

plusOne.value++ // ошибка
```

Кроме того, можно передать объект с функциями `get` и `set` для создания изменяемого ref объекта.

```js
const count = ref(1)
const plusOne = computed({
  get: () => count.value + 1,
  set: val => {
    count.value = val - 1
  }
})

plusOne.value = 1
console.log(count.value) // 0
```

## `watchEffect`

Для применения и _автоматического повторного применения_ побочного эффекта, который основан на реактивном состоянии, можно использовать метод `watchEffect`. Он запускает функцию немедленно при отслеживании своих зависимостей и повторно запустит её при изменении одной из зависимостей.

```js
const count = ref(0)

watchEffect(() => console.log(count.value))
// -> выведет в консоль 0

setTimeout(() => {
  count.value++
  // -> выведет в консоль 1
}, 100)
```

### Остановка отслеживания

Когда `watchEffect` вызывается во время работы функции [setup()](composition-api-setup.md) компонента или [хуков жизненного цикла](composition-api-lifecycle-hooks.md), то он привязывается к жизненному циклу компонента и автоматически останавливается при размонтировании компонента.

Для остальных случаев, он возвращает метод, который может быть вызван для явной остановки отслеживания:

```js
const stop = watchEffect(() => {
  /* ... */
})

// позднее
stop()
```

### Аннулирование побочных эффектов

Иногда функция наблюдателя может выполнять асинхронные побочные эффекты, которые требует дополнительных действий, при их аннулировании (т.е. в случаях, когда состояние изменилось до того, как эффекты завершились). Функция эффекта принимает функцию `onInvalidate`, которая будет использована для аннулирования действий и вызывается:

- когда эффект будет вскоре запущен повторно
- когда наблюдатель остановлен (т.е. когда компонент размонтирован, если `watchEffect` используется внутри `setup()` или хука жизненного цикла)

```js
watchEffect(onInvalidate => {
  const token = performAsyncOperation(id.value)
  
  onInvalidate(() => {
    // id был изменён или наблюдатель остановлен.
    // аннулирование выполняемой асинхронной операции
    token.cancel()
  })
})
```

Коллбэк для аннулирования регистрируется передачей функции внутрь, а не возвращением её из коллбэка, потому что возвращаемое значение важно для обработки асинхронных ошибок. Очень часто функция эффекта будет асинхронной при операциях загрузки данных:

```js
const data = ref(null)

watchEffect(async (onInvalidate) => {
  onInvalidate(() => { /* ... */ }) // регистрируем функцию перед разрешением Promise
  data.value = await fetchData(props.id)
})
```

Асинхронная функция неявно возвращает Promise, но функцию для очистки необходимо зарегистрировать перед тем, как разрешится Promise. Кроме того, Vue полагается на возвращаемый Promise для автоматической обработки потенциальных ошибок в цепочке Promise.

### Синхронизация времени очистки эффектов

Система реактивности Vue буферизирует аннулированные эффекты и выполняет их очистку асинхронно. Это сделано для избежания дублирующих вызовов, когда в одном «тике» происходит много изменений состояния. Внутренняя функция `update` компонента также является эффектом. Когда пользовательский эффект добавляется в очередь, то по умолчанию он будет вызываться **перед** всеми эффектами `update` компонента:

```html
<template>
  <div>{{ count }}</div>
</template>

<script>
  export default {
    setup() {
      const count = ref(0)

      watchEffect(() => {
        console.log(count.value)
      })

      return {
        count
      }
    }
  }
</script>
```

В этом примере:

- Счётчик будет выведен в консоль синхронно при первом запуске.
- При изменениях `count`, коллбэк будет вызываться **перед** обновлением компонента.

В случаях, когда эффект наблюдателя требуется повторно запустить **после** обновления компонента (например, при работе [со ссылками на элемента шаблона](composition-api-template-refs.md#отслеживание-ссылок-на-элементы-шаблона)), можно передать дополнительный объект настроек с опцией `flush` (значение по умолчанию — `'pre'`):

```js
// Вызовется после обновления компонента,
// поэтому можно получить доступ к обновлённому DOM
// Примечание: это также отложит первоначальный запуск эффекта
// до тех пор, пока первая отрисовка компонента не будет завершена.
watchEffect(
  () => {
    /* ... */
  },
  {
    flush: 'post'
  }
)
```

Опция `flush` может также принимать значение `'sync'`, которое принудительно заставит эффект всегда срабатывать синхронно. Однако это неэффективно и должно использоваться крайне редко.

### Отладка наблюдателей

Опции `onTrack` и `onTrigger` можно использовать для отладки поведения наблюдателя.

- `onTrack` вызывается, когда реактивное свойство или ссылка начинает отслеживаться как зависимость.
- `onTrigger` вызывается, когда коллбэк наблюдателя вызван изменением зависимости.

Оба коллбэка получают событие отладчика с информацией о зависимости, о которой идёт речь. Рекомендуем указывать `debugger` в этих коллбэках для удобного инспектирования зависимости:

```js
watchEffect(
  () => {
    /* побочный эффект */
  },
  {
    onTrigger(e) {
      debugger
    }
  }
)
```

Опции `onTrack` и `onTrigger` работают только в режиме разработки.

## `watch`

API `watch` является точным эквивалентом свойства [watch](computed.md#методы-наблюдатели) компонента. `watch` требует наблюдения за конкретным источником данных и применяет побочные эффекты в отдельной функции коллбэка. Он также ленив по умолчанию — т.е. коллбэк вызывается только тогда, когда наблюдаемый источник изменился.

- По сравнению с [watchEffect](#watcheffect), `watch` позволяет:

  - Лениво выполнять побочные эффекты;
  - Конкретнее определять какое состояние должно вызывать перезапуск;
  - Получать доступ к предыдущему и текущему значению наблюдаемого состояния.

### Отслеживание единственного источника данных

Источником данных для наблюдателя может быть функция геттер, возвращающая значение, или непосредственно реактивная ссылка `ref`:

```js
// наблюдение за геттер-функцией
const state = reactive({ count: 0 })
watch(
  () => state.count,
  (count, prevCount) => {
    /* ... */
  }
)

// наблюдение за ref-ссылкой
const count = ref(0)
watch(count, (count, prevCount) => {
  /* ... */
})
```

### Отслеживание нескольких источников данных

Наблюдатель также может отслеживать несколько источников одновременно, используя запись с массивом:

```js
const firstName = ref('');
const lastName = ref('');

watch([firstName, lastName], (newValues, prevValues) => {
  console.log(newValues, prevValues);
})

firstName.value = "John"; // выведет в консоль: ["John",""] ["", ""]
lastName.value = "Smith"; // выведет в консоль: ["John", "Smith"] ["John", ""]
```

### Отслеживание реактивных объектов

Использование наблюдателя для сравнения значений массива или объекта, которые являются реактивными, требуют, чтобы у него была копия, состоящая только из значений.

```js
const numbers = reactive([1, 2, 3, 4])

watch(
  () => [...numbers],
  (numbers, prevNumbers) => {
    console.log(numbers, prevNumbers);
  })

numbers.push(5) // Выведет в консоль: [1,2,3,4,5] [1,2,3,4]
```

Чтобы отслеживать изменения свойств в глубоко вложенном объекте или массиве нужно установить опцию `deep` в значение `true`:

```js
const state = reactive({ 
  id: 1, 
  attributes: { 
    name: "",
  },
});

watch(
  () => state,
  (state, prevState) => {
    console.log(
      "без опции deep ",
      state.attributes.name,
      prevState.attributes.name
    );
  }
);

watch(
  () => state,
  (state, prevState) => {
    console.log(
      "с опцией deep ",
      state.attributes.name,
      prevState.attributes.name
    );
  },
  { deep: true }
);

state.attributes.name = "Alex"; // Logs: "deep " "Alex" "Alex"
```

Однако, при отслеживании реактивного объекта или массива будет всегда возвращаться одна ссылка на текущее значение этого объекта как для текущего, так и для предыдущего состояния. Для полноценного отслеживания глубоко вложенных объектов или массивов, может потребоваться создание глубокой копии значений. Это может сделать с помощью утилиты, такой как [lodash.cloneDeep](https://lodash.com/docs/4.17.15#cloneDeep)

```js
import _ from 'lodash';

const state = reactive({
  id: 1,
  attributes: {
    name: "",
  },
});

watch(
  () => _.cloneDeep(state),
  (state, prevState) => {
    console.log(
      state.attributes.name, 
      prevState.attributes.name
    );
  }
);

state.attributes.name = "Alex"; // Выведет в консоль: "Alex" ""
```

### Общее поведение с `watchEffect`

Общее поведение `watch` и [`watchEffect`](#watcheffect) будет в возможностях [остановки отслеживания](#остановка-отслеживания), [аннулировании побочных эффектов](#аннулирование-побочных-эффектов) (с передачей коллбэка `onInvalidate` третьим аргументом), [синхронизации времени очистки эффектов](#синхронизация-времени-очистки-эффектов) и инструментов [отладки](#отладка-наблюдателеи).
