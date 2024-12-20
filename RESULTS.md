# Теория  
В данной работе применяется метод PINN для решения нелинейного дифференциального уравнения второго порядка, а полученный результат сравнивается с аналитическим решением. Исследуется влияение методов балансировки весов на качество обучения.
### Уравнение
Рассматривается обобщённое уравнение Шрёдингера в нелинейной среде (generalized Schrodinger equation with a dual-power law nonlinear medium):  

$$iq_t + q_{xx} + |q|^2 q (1 - \alpha |q|^2 + \beta |q|^4) = 0$$

В [[1]](#обзор-литературы) найдено аналитическое решение такого уравнения:  

$$q(x, t)=\Bigg[\frac{\left(4 k^{2} - 4 w\right) e^{\sqrt{4 k^{2} - 4 w} \left(- 2 k t + x - x_0\right)}}{- \frac{\alpha}{3} \left(4 k^{2} - 4 w\right) e^{2 \sqrt{4 k^{2} - 4 w} \left(- 2 k t + x - x_0\right)} + \left(\frac{1}{2} e^{\sqrt{4 k^{2} - 4 w} \left(- 2 k t + x - x_0\right)} + 1\right)^{2}}\Bigg]^{\frac{1}{2}}  e^{i \left(k x - w t - \theta_{0}\right)},$$

рассматривается  случай $\quad \alpha = 1,\quad \beta = 0,\quad x_0 = -55,\quad$ при котором $\quad k = 1,\quad \theta_0 = 0,\quad w = \frac{8}{9}.$
### PINN
Решение уравнения находится в виде нейросети. Это возможно потому, что нейросеть рассматривается как функция, а от функции можно считать производные разных порядков и из них составить исходное уравнение. Полученное уравнение будет использоваться как первое слагаемое `loss` (далее обозначается как `loss_f`). Задачей оптимизатора будет как можно сильнее уменьшить `loss`, а значит и заставить нейросеть удовлетворять уравнению. Чтобы в итоге не получалось тривиальное решение, вторым слагаемым `loss` будет ошибка выполнения начальных и граничных условий (далее обозначается как `loss_uv`). В данной работе начальное условие определяется как значения аналитического решения в $(x,t_0)$, а граничные условия считаются нулевыми в $(x_0,t)$ и $(x_1,t)$. В итоге минимизируется следующая функция: `loss = loss_f + loss_uv`. Весь процесс изображён на картинке:  
<img src="https://github.com/mikhakuv/PINNs_for_article/blob/main/pictures/illustration.png">  
Такой подход называется **P**hysics **I**nformed **N**eural **N**etwork и был впервые представлен в [[2]](#обзор-литературы). Он существенно отличается от численных методов тем, что в итоге получается не массив чисел, а дифференцируемая функция.  
### Балансировка весов
Несмотря на то, что метод работает неплохо, есть много попыток улучшить его. Один из наиболее распространённых приёмов - динамическая балансировка весов, она используется не только в PINN, но и в других областях. Идея заключается в том, чтобы нейросеть одинаково хорошо училась удовлетворять всем условиям. Для этого при слагаемых `loss` добавляются коэффициенты и в итоге он имеет вид: $$loss = \sum\limits_{i} lambda_i * loss_i$$ Есть несколько методов вычисления коэффициентов в процессе обучения, в данной работе изучается и совершенствуется метод SoftAdapt, предложенный в статье [[3]](#обзор-литературы). Изначально он применялся к задаче реконструкции изображений и использовал относительность прогресса в уменьшении составных частей loss. Обозначим как $loss_i(iter)$ значение $i$-того слагаемого `loss` на шаге $iter$. Тогда коэффициент $lambda_i$ при слагаемом $loss_i$ определяется по формуле:  

$$lambda_i = \frac{exp(\frac{loss_i(iter)}{loss_i(iter-1)})}{\sum\limits_{j} exp(\frac{loss_j(iter)}{loss_j(iter-1)})}$$

Смысл такой формулы простой: во-первых коэффициент при $loss_i$ снижается, если он($loss_i$) уменьшился сильнее других и повышается в обратном случае, во-вторых сумма всех коэффициентов равна 1. Ясно, что на следующем шаге оптимизатору будет выгоднее снижать $loss_i$, у которого коэффициент $lambda_i$ больше и поэтому в процессе обучения все $loss_i$ будут снижены одинаково хорошо.  
Можно разбивать `loss` не только на 2, но и на большее число слагаемых за счёт разделения граничных и начальных условий. В данной работе будет проверена работа метода SoftAdapt на 2 и 3 слагаемых.  

Также можно усовершенствовать текущий метод, добавив ему память предыдущих итераций. Это можно сделать, если присваивать значения коэффициентов $lambda_i$ следующим образом:  

$$lambda_i = \tau*lambda_i(iter-1) + (1-\tau)*lambda_i(iter)$$

где $\tau \in[0,1]$ - коэффициент, обозначающий затухание. Чем ближе $\tau$ к 1, тем больше влияние предыдущих значений $lambda_i$.  

Ещё одним улучшением является метод ReLoBRaLo, предложенный в статье [[4]](#обзор-литературы). Он учитывает изменение `loss` не только по сравнению с предыдущим шагом, но и с самым началом обучения. У него также есть память предыдущих итераций как в предыдущем методе. Коэффициенты определяются следующим образом:  
$$lambda_i = \tau*(\rho*lambda_i(iter-1) + (1-\rho) *\widehat{lambda_i}(iter)) + (1-\tau)*lambda_i(iter)$$

где $\tau \in[0,1]$ - коэффициент, обозначающий затухание, $\rho \in[0,1]$ - случайное число, генерируется каждую итерацию, а $\widehat{lambda_i}$ вычисляется по формуле:  

$$\widehat{lambda_i} = \frac{exp(\frac{loss_i(iter)}{loss_i(0)})}{\sum\limits_{j} exp(\frac{loss_j(iter)}{loss_j(0)})}$$

# Методика измерений и результаты  
### Параметры нейросети
В опытах использовалась нейросеть с топологией [2,100,100,100,100,2]: 2 входа - переменные $x$ и $t$; дальше идут 4 полносвязных слоя по 100 нейронов в каждом; 2 выхода - действительная($u$) и мнимая($v$) части. Обучение происходит в 2 этапа: сначала 300000 итераций оптимизатора Adam с темпом обучения, затухающим по закону `lr=0,999*lr` каждые 100 итераций, потом включается LBFGS. Функция активации нейронов - sin. Используемые параметры получены методом перебора как наиболее подходящие.
### Методика вычисления ошибок
Мерой успеха считалось значение функции $mse_q$ , вычисляемое как:  

$$mse_q=\frac{\sum\limits_{i=1}^n(\sqrt{u_{truth}(x_i,t_i)^2 +v_{truth}(x_i,t_i)^2} - \sqrt{u_{pred}(x_i,t_i)^2 +v_{pred}(x_i,t_i)^2})^2}{n}$$

где $n$ - количество рассматриваемых точек на области  
Также вычислялись $mse_u$, $mse_v$, $mse_{f_u}$, $mse_{f_v}$:

$$mse_u = \frac{\sum\limits_{i=1}^n(u_{truth}(x_i,t_i)-u_{pred}(x_i,t_i))^2}{n}$$

$$mse_v = \frac{\sum\limits_{i=1}^n(v_{truth}(x_i,t_i)-v_{pred}(x_i,t_i))^2}{n}$$

$$mse_{f_u} = \frac{\sum\limits_{i=1}^n(f_{pred_u}(x_i,t_i))^2}{n}$$

$$mse_{f_v} = \frac{\sum\limits_{i=1}^n(f_{pred_v}(x_i,t_i))^2}{n}$$

### Результаты
Опыты проводились на области $x \in [-85;85]$, $t \in [0;50]$. Был изучен каждый из предложенных методов балансировки весов:
* без балансировки - балансировка весов не используется
* 2 коэффициента, SoftAdapt - loss состоит из двух слагаемых, для нахождения коэффициентов при них используется метод SoftAdapt
* 3 коэффициента, SoftAdapt - loss состоит из трёх слагаемых, для нахождения коэффициентов при них используется метод SoftAdapt
* 3 случайных коэффициента - loss состоит из трёх слагаемых, коэффициенты при них генерируются случайным образом
* 3 коэффициента, SoftAdapt с памятью - loss состоит из трёх слагаемых, для нахождения коэффициентов при них используется усовершенствованный метод SoftAdapt с памятью предыдущих итераций
* 3 коэффициента, ReLoBRaLo - loss состоит из трёх слагаемых, для нахождения коэффициентов при них используется метод ReLoBRaLo

На графиках 1,2,3,4 изображены зависимости $mse_q(iter)$ для разных опытов, причём ось ординат представлена в логарифмическом масштабе:  

Из первого графика видно, что от увеличения числа коэффициентов метод SoftAdapt начинает работать всё хуже. Более того, кривая соответствующая опыту без балансировки коэффициентов проходит ниже, а значит SoftAdapt в данном случае только ухудшает обучение.

<p align="center"><img src="https://github.com/mikhakuv/PINNs_for_article/blob/main/pictures/results_chart1.PNG"></p>    

Второй график подтверждает то, что SoftAdapt в данном случае мешает обучению, ведь опыт со случайными коэффициентами оказывается более успешным. Но незначительная модификация SoftAdapt, имеющая память предыдущих итераций, всё-таки даёт результаты лучше, чем опыт со случайными коэффициентами.  

<p align="center"><img src="https://github.com/mikhakuv/PINNs_for_article/blob/main/pictures/results_chart2.PNG"></p>    

Третий график сравнивает лучшие кривые из графиков 1 и 2 с методом ReLoBRaLo. Отчётливо видно, что он обходит оба. Также можно заметить, что опыт, использующий SoftAdapt с памятью коэффициентов, лишь немного уступает в качестве опыту с отсутствием балансировки.  

<p align="center"><img src="https://github.com/mikhakuv/PINNs_for_article/blob/main/pictures/results_chart3.PNG"></p>    

На четвёртом графике собраны кривые со всех опытов.  

<p align="center"><img src="https://github.com/mikhakuv/PINNs_for_article/blob/main/pictures/results_chart4.PNG"></p>   

На графике 5 изображены $mse_q$ каждого из опытов после окончания обучения:  

<p align="center"><img src="https://github.com/mikhakuv/PINNs_for_article/blob/main/pictures/results_chart5.PNG"></p>  

Из представленных выше графиков видно, что метод ReLoBRaLo с тремя коэффициентами позволяет получить наилучшие результаты. Ниже представлены результаты обучения нейросети таким методом:  
Метрики: $mse_u: 4.272095 *10^{-06}, mse_v: 4.305910 *10^{-06}, mse_q: 6.840848 *10^{-07}, mse_{f_u}: 2.922648 *10^{-09}, mse_{f_v}: 2.940519 *10^{-09}$  
Полученное решение(_pred) в сравнении с аналитическим(_truth):  

<p align="center"><img src="https://github.com/mikhakuv/PINNs_for_article/blob/main/pictures/exp6_results_u.PNG"><img src="https://github.com/mikhakuv/PINNs_for_article/blob/main/pictures/exp6_results_v.PNG"></p>  

Модули полученного и аналитического решений в срезах по $t$(красный и зелёный соответственно):  

<p align="center"><img src="https://github.com/mikhakuv/PINNs_for_article/blob/main/pictures/exp6_results_q.PNG"></p>  

Разность модулей полученного и аналитического решений:  

<p align="center"><img src="https://github.com/mikhakuv/PINNs_for_article/blob/main/pictures/exp6_results_q_diff.PNG"></p>  

Зависимость качества выполнения условий уравнения от $t$(чем ближе к 0, тем лучше):  

<p align="center"><img src="https://github.com/mikhakuv/PINNs_for_article/blob/main/pictures/exp6_results_mean.PNG"></p>  

Как менялась действительная часть выдачи нейросети в процессе обучения:  

<p align="center"><img src="https://github.com/mikhakuv/PINNs_for_article/blob/main/pictures/train_process.gif"></p>  

Статистика по всем проведённым экспериментам и данные для построения графиков находятся в файлах:
[performance_table.xlsx](https://github.com/mikhakuv/PINNs_Loss_Balancing/blob/main/statistics/performance_table.xlsx),
[stats.xlsx](https://github.com/mikhakuv/PINNs_Loss_Balancing/blob/main/statistics/stats.xlsx)  

### Вывод
В данной работе изучалась эффективность методов балансировки коэффициентов при решении обобщённого уравнения Шрёдингера в нелинейной среде. Проведённые эксперименты показали, что ни SoftAdapt ни его улучшенная версия с памятью коэффициентов не дают никакой выгоды по сравнению с отсутствием балансировки. Наоборот, они замедляют обучение и ухудшают результаты. В то же время метод ReLoBRaLo даёт некоторое преимущество и позволяет достигнуть очень высокой точности в решении рассмотренного уравнения.  
# Обзор Литературы  
1. !!!статья с уравнением
2. *Maziar Raissi, Paris Perdikaris, George Em Karniadakis* "Physics Informed Deep Learning (Part I): Data-driven Solutions of Nonlinear Partial Differential Equations"
3. *A. Ali Heydari, Craig A. Thompson, Asif Mehmood* "SoftAdapt: Techniques for Adaptive Loss Weighting of Neural Networks with Multi-Part Loss Functions"
4. *Rafael Bischof, Michael Kraus* "Multi-Objective Loss Balancing for Physics-Informed Deep Learning"
