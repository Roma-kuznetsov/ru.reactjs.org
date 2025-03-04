---
id: reconciliation
title: Согласование
permalink: docs/reconciliation.html
---

React предоставляет декларативный API, который позволяет не беспокоиться о том, что именно изменяется при каждом обновлении. Благодаря этому, писать приложения становится намного проще, но может быть неочевидно как именно это реализовано внутри React. В этой статье объясняются решения, принятые нами для алгоритма сравнения в React, которые делают обновления компонента предсказуемыми, и в то же время достаточно быстрыми для высокопроизводительных приложений.

## Мотивация {#motivation}

При работе с React вы понимаете `render()` как функцию, которая создаёт дерево React-элементов в какой-то момент времени. При следующем обновлении состояния или пропсов функция `render()` возвращает новое дерево React-элементов. Во время выполнения этой функции React должен выяснить как наиболее эффективно обновить пользовательский интерфейс для соответствия новому дереву.

Существует несколько общих решений алгоритмической проблемы трансформации одного дерева в другое за минимальное количество операций. Тем не менее, даже [передовые алгоритмы](http://grfia.dlsi.ua.es/ml/algorithms/references/editsurvey_bille.pdf) имеют сложность порядка O(n<sup>3</sup>), где n — это число элементов в дереве.

Если бы мы использовали это в React, отображение 1000 элементов потребовало бы порядка миллиарда сравнений. Это слишком дорого. Взамен, React реализует эвристический алгоритм O(n), который основывается на двух утверждениях:

1. Два элемента с разными типами произведут разные деревья.
2. Разработчик может указать какие дочерние элементы останутся стабильными между рендерами с помощью пропа `key`.

На практике эти утверждения верны почти во всех случаях.

## Алгоритм сравнения {#the-diffing-algorithm}

React сравнивает деревья начиная с их корневых элементов и направляется вниз. Сравниваются типы (теги) корневых элементов.

### Элементы различных типов {#elements-of-different-types}

Всякий раз, когда корневые элементы имеют различные типы, React уничтожает старое дерево и строит новое с нуля. Трансформация из `<a>` в `<img>`, или из `<Article>` в `<Comment>`, или из `<Button>` в `<div>` приведут к полному перестроению вложенных элементов.

При уничтожении дерева старые DOM-узлы удаляются. Экземпляры компонента получают `componentWillUnmount()`. При построении нового дерева, новые DOM-узлы вставляются в DOM. Экземпляры компонента получают `UNSAFE_componentWillMount()`, а затем `componentDidMount()`. Любое состояние, связанное со старым деревом, теряется.

Любые компоненты, лежащие ниже корневого, также размонтируются, а их состояние уничтожится. Например, это произойдёт при таком сравнении: 

```html
<div>
  <Counter />
</div>

<span>
  <Counter />
</span>
```

При этом старый `Counter` уничтожится и смонтируется новый.

>Примечание:
>
>Следующие методы объявлены устаревшими, поэтому в новом коде их [не стоит](/blog/2018/03/27/update-on-async-rendering.html) использовать:
>
>- `UNSAFE_componentWillMount()`

### DOM-элементы одного типа {#dom-elements-of-the-same-type}

При сравнении двух React DOM-элементов одного типа, React смотрит на атрибуты обоих, сохраняет лежащий в основе этих элементов DOM-узел и обновляет только изменённые атрибуты. Например:

```html
<div className="before" title="stuff" />

<div className="after" title="stuff" />
```

Сравнивая эти элементы, React знает, что нужно модифицировать только `className` у DOM-узла.

Обновляя `style`, React также знает, что нужно обновлять только изменившиеся свойства. Например: 

```html
<div style={{color: 'red', fontWeight: 'bold'}} />

<div style={{color: 'green', fontWeight: 'bold'}} />
```

При конвертации между этими элементами, React знает, что нужно модифицировать только стиль `color`, не затронув `fontWeight`.

После обработки DOM-узла React рекурсивно проходит по дочерним элементам.

### Компоненты одного типа {#component-elements-of-the-same-type}

Когда компонент обновляется, его экземпляр остаётся прежним, поэтому его состояние сохраняется между рендерами. React обновляет пропсы базового экземпляра компонента для соответствия новому элементу и вызывает `UNSAFE_componentWillReceiveProps()`, `UNSAFE_componentWillUpdate` и `componentDidUpdate()` на базовом экземпляре.

Далее вызывается метод `render()` и алгоритм сравнения рекурсивно обходит предыдущий и новый результаты.

>Примечание:
>
>Следующие методы объявлены устаревшими, поэтому в новом коде их [не стоит](/blog/2018/03/27/update-on-async-rendering.html) использовать:
>
>- `UNSAFE_componentWillUpdate()`
>- `UNSAFE_componentWillReceiveProps()`

### Рекурсия по дочерним элементам {#recursing-on-children}

По умолчанию при рекурсивном обходе дочерних элементов DOM-узла React одновременно проходит по обоим спискам потомков и создаёт мутацию, когда находит отличие.

Например, при добавлении элемента в конец дочерних элементов, преобразование между этими деревьями работает отлично: 

```html
<ul>
  <li>первый</li>
  <li>второй</li>
</ul>

<ul>
  <li>первый</li>
  <li>второй</li>
  <li>третий</li>
</ul>
```

React сравнит два дерева `<li>первый</li>`, сравнит два дерева `<li>второй</li>`, а затем вставит дерево `<li>третий</li>`.

При вставке элемента в начало, прямолинейная реализация такого алгоритма будет работать не эффективно. Например, преобразование между этими деревьями работает плохо: 

```html
<ul>
  <li>Санкт-Петербург</li>
  <li>Москва</li>
</ul>

<ul>
  <li>Ростов-на-Дону</li>
  <li>Санкт-Петербург</li>
  <li>Москва</li>
</ul>
```
React, вместо того чтобы оставить `<li>Санкт-Петербург</li>`  и `<li>Москва</li>` нетронутыми, будет мутировать каждого потомка. Эта неэффективность может стать проблемой.

### Ключи {#keys}

Для решения этой проблемы React поддерживает атрибут `key`. Когда у дочерних элементов есть ключи, React воспользуется ими, чтобы сопоставить потомков исходного дерева с потомками последующего дерева. Например, если добавить `key` к неэффективному примеру выше, преобразование дерева станет эффективнее:

```html
<ul>
  <li key="2015">Санкт-Петербург</li>
  <li key="2016">Москва</li>
</ul>

<ul>
  <li key="2014">Ростов-на-Дону</li>
  <li key="2015">Санкт-Петербург</li>
  <li key="2016">Москва</li>
</ul>
```

Теперь React знает, что элемент с ключом `'2014'` — новый, а элементы с ключами `'2015'` и `'2016'` переместились.

На практике найти ключ обычно несложно. Элемент, который вы хотите отобразить, уже может иметь уникальный идентификатор, и ключ может быть взят из ваших данных:

```js
<li key={item.id}>{item.name}</li>
```

Когда уникальное значение отсутствует, вы можете добавить новое свойство идентификатора в вашу модель или прохешировать данные, чтобы сгенерировать ключ. Ключ должен быть уникальным только среди его соседей, а не глобально.

В крайнем случае вы можете передать индекс элемента массива в качестве ключа. Это работает хорошо в случае, если элементы никогда не меняют порядок. Перестановки элементов вызывают замедление.

Вдобавок перестановки элементов могут вызвать проблемы с состоянием компонента, когда в качестве ключей используются индексы. Экземпляры компонента обновляются и повторно используются на основе своих ключей. Если ключ является индексом, то перемещение элемента изменяет его. В результате состояние компонента для таких элементов , как неуправляемые `<input>`, может смешаться и обновиться неожиданным образом.

На CodePen [есть примеры проблем, которые могут быть вызваны использованием индексов в качестве ключей](codepen://reconciliation/index-used-as-key), а также есть [обновлённая версия того же примера, которая показывает как решаются проблемы с перестановкой, сортировкой и вставкой элементов в начало, если не использовать индексы как ключи](codepen://reconciliation/no-index-used-as-key).

## Компромиссы {#tradeoffs}

Важно помнить, что алгоритм согласования — это деталь реализации. React может повторно рендерить всё приложение на каждое действие, конечный результат будет тем же. Для ясности, повторный рендер в этом контексте означает вызов функции `render` для всех компонентов, но это не означает, что React размонтирует и смонтирует их заново. Он лишь применит различия следуя правилам, которые были обозначены выше.

    Мы регулярно совершенствуем эвристику, чтобы повысить производительность. В текущей реализации вы можете выразить факт того, что поддерево было перемещено среди своих соседей, но вы не можете сказать, что оно переместилось куда-то в другое место. Алгоритм повторно рендерит всё поддерево.

React полагается на эвристику, следовательно, если утверждения, на которых она основана, не соблюдены, то пострадает производительность. 

1. Алгоритм не будет пытаться сравнивать поддеревья компонентов разных типов. Если вы заметите за собой, что пытаетесь использовать компоненты разных типов с очень схожим выводом, то, вероятно, стоит их сделать компонентами одного типа. На практике мы не выявили с этим проблем.

2. Ключи должны быть стабильными, предсказуемыми и уникальными. Нестабильные ключи (например, произведённые с помощью `Math.random()`) вызовут необязательное пересоздание многих экземпляров компонента и DOM-узлов, что может вызывать ухудшение производительности и потерю состояния у дочерних компонентов.
