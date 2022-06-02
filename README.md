# Тестовое задание KazanExpress

## Задача

В маркетплейс каждый день поступает множество новых товаров и каждый из них необходимо
отнести в определенную категорию в дереве категорий. На это тратится много сил и времени, 
поэтому необходимо предсказывать категорию на основе названий и параметров товаров.

## Метрика
Предсказывать листовую категорию товара может показаться тривиальной задачей, однако нам важен путь к листовой категории. 
Для неверно предсказанных листовых категорий мы хотим добавлять скор, если были угаданы предшествующие родительские категории на разных уровнях.
Для честной оценки многоуровневой классификации мы будем использовать видоизмененную метрику **F1**, взвешенную на размер 
выборки класса(при подсчете учитываются только листовые категории):

<img src="https://github.com/ShirokovSe/Hierarchial-classifier_for_products_categorization/blob/main/additional/F1-score.png" width="750">

## Данные

Данные для задачи находятся в папке additional.

### Формат входные данных

**categories_tree.csv** - файл с деревом категорий на маркетплейсе. У каждой категории есть id, 
заголовок и parent_id, по которому можно восстановить полный путь категории.

**train.parquet** - файл с товарами на маркетплейсе. 
У каждого товара есть:
<lu>
  <li><i>id</i>- идентификатор товара</li>
  <li><i>title</i> - заголовок</li>
  <li><i>short_description</i> - краткое описание</li>
  <li><i>name_value_characteristics</i> - название:значение характеристики товара, 
    может быть несколько для одного товара и для одной характеристик</li>
  <li><i>rating</i> - средний рейтинг товара</li>
  <li><i>feedback_quantity</i> - количество отзывов по товару</li>
  <li><i>category_id</i> - категория товара(таргет)
</lu>

**test.parquet** - файл идентичный train.parquet, но без реального category_id, именно его
вам и предстоит предсказать.

## Наблюдения, идеи

### Дисбаланс классов

Первое, на что я обратил внимание - это существенный дисбаланс классов. Попытка сделать Downsample, то есть уменьшить число 
экземпляров мажоритарных классов ожидаемо не дало улучшений в работе моделей, а также привело к существенной потере информации.
Следующая идея была применение Upsample, соответсвенно, просто увеличить число экземляров минорных классов, однако, и это не дало
никаких изменений. Последняя идея для выравнивания классов, которую я пробовал применить состояла в следующем: выбрать корпуса
текстов для каждой категории и случайным образом формировать новые названия товаров из всего корпуса слов до выравнивания с числом 
экземпляров мажоритарного класса, относящихся к выбранной категории. Возникшие проблемы: датасет стал настолько огромным, что перестал
помещаться в память, уменьшение числа дополнительных экземпляров приводит к очень долгому обучению, которые не дало никаких значимых 
результатов, также еще одна проблема заключалась в том, что некоторые классы представлепны 2-5 экземплярами с малым числом слов, что
привело просто к дублирвоанию экземляров. На этом я решил оставить идею выравнивания классов. Также сыграло роль общее число классов - 1231.

### Объединение моделей

В процессе обучения модели fasttext, я попробовал предсказывать не крайнюю категорию, а по всему корпусу по порядку определять все категории - предки. 
Данный подход дал интересные результаты, модель fasttext (пробовал только на ней) с высокой точностью предсказывала родительские категории, например, 
первого родителя 0 или -1 (в случае вложенности меньше 6) с точностью 99.8%, второго родителя примерно 99,8 и т.д. Последняя
родительская категория предсказывалась с точностью 88,5%. Тогда я предположил, а почему бы не объединить работу моделей, с помощью fasttext 
предсказывать дополнительные признаки для линейных моделей (все родительские классы), затем стандартизировать, объединять с векторным представлением
и передавать на вход линейной модели. В результате, получилось повысить качество работы линейной модели примерно на 1%, что не дало результат лучше, чем
работа fasttext.

### Отедльная обработка слов-приложений

При выполнении задания обратил внимание на интересный факт, несмотря на высокие показатели работы библиотек для обработки текста, ни один из методов практически не обрабатывает слова - приложения (например, красно-белое), и 
преврещает их в одно слово (например, краснобелое). поэтому предварительно, слова такого рода были токенизрованы отдельно. Такое решение позволилио увеличить качество модели на итоговой метрике на 1,2%

### Применение тяжелых нейронных сетей (BERT)

Также пробовал применить BERT, но обучение одной эпохи в среднем занимало около 4-5 часов на GPU без значительных результатов. Поэтому я посчитал, что
для данного задания применение такие нейроннызх сетей нецелесообразно, время работы больше, чем у линейных моделей или fasttext, а полученное качество не является
наилучшим.

## Итог

В процессе работы были опробованы 5 различных подходов. Они рассмотрены в траблице ниже с соответствующе рассчитанной метрикой.
<table align='center'>
                <tr><th>fasttext</th><th>Multinominal Bayes</th><th>word2vec + logistic regression</th><th>word2vec+NaiveBayes</th><th>word2vec+SVM</th><th>word2vec + basic NeuralNetwork</th><th>TF-IDF + logistic regression</th></tr>
                <tr><td>0.943</td><td>0.884</td><td>0.916</td><td>0.867</td><td>0.867</td><td>0.928</td><td>0.67</td>
                </table>
                
# Заключение

Таким образом, в результате работы была выбрана модель fasttext с подбором параметров, применение которой позволило получить на ивысшие результаты. Для обработки текста наилучшие 
результаты были получены при лемматизации и отдельной обработки слов-приложений.
    
