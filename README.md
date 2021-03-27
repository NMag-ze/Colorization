# Colorization (Окрашивание ч/б фотографий)

**Основная логика**

**Проект делали:
Федчун Никита
Латипов Руслан**

В этом разделе рассмотрю рендеринг изображения, теорию цифрового цвета и основную логику нейросети.

Чёрно-белые изображения можно представить в виде сетки из пикселей. У каждого пикселя есть значение яркости, лежащее в диапазоне от 0 до 255, от чёрного до белого. 

![](https://habrastorage.org/webt/9g/ss/xl/9gssxlwzxzloxwvc6bsyql1sjwg.png)
  
Цветные изображения состоят из трёх слоёв: красного, зелёного и синего. Допустим, нужно разложить по трём каналам картинку с зелёным листиком на белом фоне. Вы можете подумать, что листик будет представлен только в зелёном слое. Но, как видите, он есть во всех трёх слоях, потому что слои определяют не только цвет, но и яркость.
 ![](https://habrastorage.org/webt/qa/uo/ai/qauoaimgzzpkuocemsvhu7uosys.png)
 
К примеру, чтобы получить белый цвет, нам нужно получить равное распределение всех цветов. Если добавить одинаковое количество красного и синего, то зелёный станет ярче. То есть в цветном изображении с помощью трёх слоёв кодируется цвет и контрастность.  
![](https://habrastorage.org/webt/7m/v4/o0/7mv4o0fsqc0ekox1havhpcckev8.png)

Как и в чёрно-белом изображении, пиксели каждого слоя цветного изображения содержат значение от 0 до 255. Ноль означает, что у этого пикселя в данном слое нет цвета. Если во всех трёх каналах стоят нули, то в результате на картинке получается чёрный пиксель.

Нейросеть устанавливает взаимосвязь между входным и выходным значениями. В нашем случае нейросеть должна найти связующие черты между чёрно-белыми и цветными изображениями. То есть мы ищем свойства, по которым можно сопоставить значения из чёрно-белой сетки со значениями из трёх цветных.
 
![](https://habrastorage.org/webt/1k/zl/hu/1kzlhuovpv7ovlq9n7lsxjozxbk.png) 
f() — нейросеть, [B&W] — входные данные, [R],[G],[B] — выходные данные.

**Технические пояснения**

Напомню, что на входе у нас сетка, представляющая чёрно-белое изображение. А на выходе — две сетки со значениями цветов. Между входными и выходными значениями мы создали связующие фильтры. У нас получилась свёрточная нейросеть.

Для обучения сети используются цветные изображения. Мы преобразовали из цветового пространства RGB в Lab. Чёрно-белый слой подаётся на вход, а на выходе получаются два раскрашенных слоя.
 
![](https://habrastorage.org/webt/tj/lw/0z/tjlw0zbxhqky-geusbsdpfkzf4y.png)

Мы в одном диапазоне сопоставляем (map) вычисленные значения с реальными, тем самым сравнивая их друг с другом. Границы диапазона от —1 до 1. Для сопоставления вычисленных значений мы используем функцию активации tanh (гиперболическая тангенциальная). Если применить её к какому-нибудь значению, то функция вернёт значение в диапазоне от —1 до 1.

Реальные значения цветов меняются от —128 до 128. В пространстве Lab это диапазон по умолчанию. Если каждое значение разделить на 128, то все они окажутся в границах от —1 до 1. Такая «нормализация» позволяет сравнивать погрешность нашего вычисления.

После вычисления результирующей погрешности нейросеть обновляет фильтры, чтобы скорректировать результат следующей итерации. Вся процедура повторяется циклически, пока погрешность не станет минимальной.
Давайте разберёмся с синтаксисом этого кода:

X = rgb2lab(1.0/255*image)[:,:,0]
Y = rgb2lab(1.0/255*image)[:,:,1:]

1.0/255 означает, что мы используем 24-битное цветовое пространство RGB. То есть для каждого цветового канала мы используем значения в диапазоне от 0 до 255. Это даёт нам 16,7 миллиона цветов.

Но поскольку человеческий глаз может распознавать лишь от 2 до 10 млн цветов, то использовать более широкое цветовое пространство не имеет смысла.

Y = Y / 128

Цветовое пространство Lab использует другой диапазон. Цветовой спектр ab варьируется от —128 до 128. Если поделить все значения выходного слоя на 128, то они уложатся в дипазон от —1 до 1, и тогда можно будет сопоставить эти значения с теми, что вычислила наша нейросеть.

После того, как с помощью функции rgb2lab() преобразовали цветовое пространство, мы с помощью [:,:, 0] выбираем чёрно-белый слой. Это входные данные для нейросети. [:,:, 1: ] выбирает два цветных слоя, красно-зелёный и сине-жёлтый.

После обучения нейросети выполняем последнее вычисление, которое преобразуем в картинку.

output = model.predict(X)
output = output * 128

Здесь мы подаём на вход чёрно-белое изображение и прогоняем его через обученную нейросеть. Берём все выходные значения от —1 до 1 и умножаем их на 128. Так мы получаем корректные цвета в системе Lab.

canvas = np.zeros((400, 400, 3))
canvas[:,:,0] = X[0][:,:,0]
canvas[:,:,1:] = output[0]

Создаём чёрный RGB-холст, заполнив все три слоя нулями. Затем копируем чёрно-белый слой из тестового изображения и добавляем два цветных слоя. Получившийся массив значений пикселей преобразуем в изображение.
