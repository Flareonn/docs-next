# Механизмы отрисовки и оптимизации

> Это необязательный раздел для чтения, чтобы узнать как хорошо использовать Vue, но он предоставляет дополнительную информацию, если интересно понять как работает отрисовка страниц под капотом.

## Виртуальный DOM

Сейчас, когда знаем как методы-наблюдатели обновляют компоненты, может возникнуть вопрос: каким образом эти изменения попадают в конечном итоге в DOM?! Возможно, ранее вы уже слышали о виртуальном DOM, многие фреймворки, включая и Vue, используют эту парадигму чтобы убедиться что интерфейсы эффективно отражают все изменения, выполненные в JavaScript.

<div class="reactivecontent">
  <common-codepen-snippet title="Как работает виртуальный DOM?" slug="KKNJKbw" tab="result" theme="light" :height="500" :editable="false" :preview="false" />
</div>

Копия DOM в JavaScript, называемая виртуальным DOM, создаётся из-за того, что работа с DOM через JavaScript является дорогостоящими вычислительными операциями. В тоже время как выполнение обновлений в JavaScript дёшево, поиск требуемых DOM-узлов и их обновление с помощью JavaScript затратно. Поэтому набирается список предстоящих изменений и применяется к DOM одновременно.

Виртуальный DOM — это легковесный объект JavaScript, создаваемый render-функцией. Он принимает три аргумента: элемент, объект с данными, входными параметрами, атрибутами и многим другим, а также массив. Массив — место где указываются дочерние элементы, каждый из которых тоже имеет по три аргумента, и они также могут иметь свои дочерние элементы и так далее, пока не будет построено полное дерево элементов.

Если требуется обновить элементы списка, мы выполняем это через JavaScript, используя реактивность о которой упоминали ранее. Затем все эти изменения вносятся в JavaScript копию, т.е. в виртуальный DOM, и выполняется операция сравнения с оригинальным DOM. Только после этого определяются какие обновления необходимо сделать. Таким образом виртуальный DOM позволяет сделать обновления UI производительными!
