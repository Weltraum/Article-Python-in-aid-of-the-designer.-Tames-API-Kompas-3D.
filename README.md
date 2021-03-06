# Python на службе у конструктора. Укрощаем API Kompas 3D
![Logo](https://habrastorage.org/files/75a/869/fe6/75a869fe65854087a5b965b1b2c82f75.png "Python конструктора")

Работая в конструкторском отделе, я столкнулся с задачей - рассчитать трудоёмкость разработки конструкторской документации. Если брать за основу документ: [«Типовые нормативы времени на разработку конструкторской документации. ШИФР 13.01.01" (утв. Минтрудом России 07.03.2014 N 003)»](http://www.consultant.ru/document/cons_doc_LAW_199653/), то для расчета трудоёмкости чертежа детали нам необходимы следующие данные:

   *   Формат чертежа и количество листов
   *   Масштаб
   *   Количество размеров на чертеже (включая знаки шероховатости и выносные линии)
   *   Количество технических требований

Из имеющихся инструментов на предприятии имеем: Kompas 3D v14 и Python 3.5.

В интернете не так много статей о написании программ с использованием API Kompas 3D, и ещё меньше информации о том, как это сделать на Python. Попробую рассказать по шагам, как решалась поставленная задача и на какие грабли приходилось наступать. Статья рассчитана на людей, владеющих основами программирования и знакомых с языком Python. Итак, приступим.

## Подготовительная операция:
Убедитесь, что на вашем компьютере установлена программа Kompas 3D, версии не ниже 14 и Python 3. Вам также необходимо установить [pywin3](http://sourceforge.net/projects/pywin32/) (Python for Windows extensions).

## Подключение к Kompas 3D:
Система Kompas 3D имеет две версии API: API5, которая предоставляет интерфейс KompasObject, и API7, предоставляющая интерфейс IKompasAPIObject. API версии 5 и 7 во многом дублируют свой функционал, но, со слов разработчиков, в 7-ой версии более выражен объектно-ориентированный подход. В данной статье акцент сделан на 7-ю версию.

Функция подключения выглядит следующим образом:

```
import pythoncom
from win32com.client import Dispatch, gencache

# Подключение к API7 программы Kompas 3D
def get_kompas_api7():
    module = gencache.EnsureModule("{69AC2981-37C0-4379-84FD-5DD2F3C0A520}", 0, 1, 0)
    api = module.IKompasAPIObject(
        Dispatch("Kompas.Application.7")._oleobj_.QueryInterface(module.IKompasAPIObject.CLSID,
                                                                 pythoncom.IID_IDispatch))
    const = gencache.EnsureModule("{75C9F5D0-B5B8-4526-8681-9903C567D2ED}", 0, 1, 0).constants
    return module, api, const
```
Чуть подробнее о модуле win32com [здесь](http://docs.activestate.com/activepython/3.4/pywin32/html/com/win32com/HTML/GeneratedSupport.html).

Теперь, чтобы подключиться к интерфейсу, нам понадобиться следующий код:

```
module7, api7, const7 = get_kompas_api7()   # Подключаемся к API7 
app7 = api7.Application                     # Получаем основной интерфейс
app7.Visible = True                         # Показываем окно пользователю (если скрыто)
app7.HideMessage = const7.ksHideMessageNo   # Отвечаем НЕТ на любые вопросы программы
print(app7.ApplicationName(FullName=True))  # Печатаем название программы
```
Для более глубокого понимания API заглянем в SDK. На моём компьютере она находится по адресу: <i>C:\Program Files\ASCON\KOMPAS-3D V16\SDK\SDK.chm</i>. Здесь можно подробнее узнать, например, о методе HideMessage:

![HideMessage](https://habrastorage.org/files/d4f/094/eb2/d4f094eb276a474b92ae63235b35e46a.png "Окно справки по SDK")

После выполнения нашего кода вернём всё на свои места: если Kompas 3D был запущен нами (в процессе работы скрипта), мы его и закроем. Самый простой способ определить, запущен ли процесс, - использовать стандартный модуль subprocess:

```
import subprocess

# Функция проверяет, запущена ли программа Kompas 3D
def is_running():
    proc_list = subprocess.Popen('tasklist /NH /FI "IMAGENAME eq KOMPAS*"',
                                 shell=False,
                                 stdout=subprocess.PIPE).communicate()[0]
    return True if proc_list else False
```

Данная функция проверяет, запущен ли процесс «KOMPAS» [стандартными методами Windows](https://technet.microsoft.com/en-us/library/bb491010.aspx#mainSection). Обратите внимание, что разные версии программы Kompas 3D могут иметь разные наименования процессов!

## Считаем количество листов и их формат:
Тут всё просто: у нашего документа doc7 имеется интерфейс коллекции листов оформления LayoutSheets. Каждый лист обладает свойством формата и кратности. Для Компаса, начиная с 15 версии, интерфейс LayoutSheets доступен не только для файлов чертежей, но и для спецификаций и текстовых документов.

```
# Посчитаем количество листов каждого из формата
def amount_sheet(doc7):
    sheets = {"A0": 0, "A1": 0, "A2": 0, "A3": 0, "A4": 0, "A5": 0}
    for sheet in range(doc7.LayoutSheets.Count):
        format = doc7.LayoutSheets.Item(sheet).Format  # sheet - номер листа, отсчёт начинается от 0
        sheets["A" + str(format.Format)] += 1 * format.FormatMultiplicity
    return sheets
```

Посмотрим на процесс изучения SDK для поиска интересующих нас функций:

![Animated](https://habrastorage.org/files/d71/883/e2c/d71883e2c3654b0fb7ca0bec92b8998c.gif)


## Читаем основную надпись:
Здесь нам поможет всё тот же LayoutSheets:

```
# Прочитаем масштаб из штампа, ячейка №6
def stamp_scale(doc7):
    stamp = doc7.LayoutSheets.Item(0).Stamp  # Item(0) указывает на штамп первого листа 
    return stamp.Text(6).Str    
```

На самом деле ячейка №6 для листа с другим оформлением может содержать не масштаб, а совсем иную информацию.  Посмотрим, как в Kompas 3D определяются стили оформления чертежа:

![Style](https://habrastorage.org/files/371/b6d/490/371b6d490efb4ce59876457318c4513d.png)

Таким образом, важно проверять, какому файлу и номеру оформления соответствует лист чертежа. Также стоит помнить, что  документ может содержать титульный лист! Поэтому придётся усложнить код. Применим [регулярные выражения](https://habrahabr.ru/post/115825/), т.к. текст в ячейке может являться ссылкой:

```
import os
import re

# Прочитаем основную надпись чертежа
def stamp(doc7):
    for sheet in range(doc7.LayoutSheets.Count):
        style_filename = os.path.basename(doc7.LayoutSheets.Item(sheet).LayoutLibraryFileName)
        style_number = int(doc7.LayoutSheets.Item(sheet).LayoutStyleNumber)

        if style_filename in ['graphic.lyt', 'Graphic.lyt'] and style_number == 1:
            stamp = doc7.LayoutSheets.Item(sheet).Stamp
            return {"Scale": re.search(r"\d+:\d+", stamp.Text(6).Str).group(),
                    "Designer": stamp.Text(110).Str}

    return {"Scale": 'Неопределенный стиль оформления',
            "Designer": 'Неопределенный стиль оформления'}
```

Остался последний вопрос: как узнать нужный номер ячейки? Для этих целей удобно создать файл чертежа, в котором интересующие нас ячейки будут заполнены, а после - прочитать все возможные варианты с помощью следующей функции:

```
# Просмотр всех ячеек
def parse_stamp(doc7, number_sheet):
    stamp = doc7.LayoutSheets.Item(number_sheet).Stamp
    for i in range(10000):
        if stamp.Text(i).Str:
            print('Номер ячейки = %-5d Значение = %s' % (i, stamp.Text(i).Str))
```

## Считаем количество пунктов технических требований:
Согласно SDK, нам всего-то нужно получить интерфейс TechnicalDemand от IDrawingDocument, а
IDrawingDocument можно получить от iDocuments с помощью замечательного метода с говорящим названием IUnknown::QueryInterface. И только в SDK 16 версии Kompas 3D появилось разъяснение, как это сделать:

![IUnknown](https://habrastorage.org/files/e7c/a58/b7e/e7ca58b7efdb49f9ace1f39ad64b7f12.png)

С такими разъяснениями легко написать следующее:

```
# Подсчет технических требований, в том случае, если включена автоматическая нумерация
def count_TT(doc7, module7):
    doc2D_s = doc7._oleobj_.QueryInterface(module7.NamesToIIDMap['IDrawingDocument'],
                                           pythoncom.IID_IDispatch)
    doc2D = module7.IDrawingDocument(doc2D_s)
    text_TT = doc2D.TechnicalDemand.Text

    count_tt = 0                                 # Количество пунктов технических требований
    for i in range(text_TT.Count):               # Проходим по каждой строчке технических требований
        if text_TT.TextLines[i].Numbering == 1:  # и проверяем, есть ли у строки нумерация
            count_tt += 1                        

    # Если нет нумерации, но есть текст
    if not count_tt and text_TT.TextLines[0]:
        count_tt += 1

    return count_tt
```

Стоит отметить, что данный код полагается на автоматическую нумерацию технических требований. Так что, если автоматическая нумерация не применялась или технические требования набраны с использованием простого инструмента «Текст», код будет сложнее. Оставляю решение данной задачи на читателя.

## Считаем количество размеров на чертеже:
При подсчёте размеров, надо иметь в виду, что необходимо посчитать их на каждом из видов чертежа:

```
# Подсчёт размеров на чертеже, для каждого вида по отдельности
def count_dimension(doc7, module7):
    IKompasDocument2D = doc7._oleobj_.QueryInterface(module7.NamesToIIDMap['IKompasDocument2D'],
                                                     pythoncom.IID_IDispatch)
    doc2D = module7.IKompasDocument2D(IKompasDocument2D)
    views = doc2D.ViewsAndLayersManager.Views

    count_dim = 0
    for i in range(views.Count):
        ISymbols2DContainer = views.View(i)._oleobj_.QueryInterface(module7.NamesToIIDMap['ISymbols2DContainer'],
                                                                    pythoncom.IID_IDispatch)
        dimensions = module7.ISymbols2DContainer(ISymbols2DContainer)

        # Складываем все необходимые размеры
        count_dim += dimensions.AngleDimensions.Count + \
                     dimensions.ArcDimensions.Count + \
                     dimensions.Bases.Count + \
                     dimensions.BreakLineDimensions.Count + \
                     dimensions.BreakRadialDimensions.Count + \
                     dimensions.DiametralDimensions.Count + \
                     dimensions.Leaders.Count + \
                     dimensions.LineDimensions.Count + \
                     dimensions.RadialDimensions.Count + \
                     dimensions.RemoteElements.Count + \
                     dimensions.Roughs.Count + \
                     dimensions.Tolerances.Count

    return count_dim
```

## Основная функция скрипта
В результате проделанной работы мы получили следующее:

```
def parse_design_documents(paths):
    is_run = is_running()                           # Установим флаг, который нам говорит, 
                                                    # запущена ли программа до запуска нашего скрипта

    module7, api7, const7 = get_kompas_api7()       # Подключаемся к программе
    app7 = api7.Application                         # Получаем основной интерфейс программы
    app7.Visible = True                             # Показываем окно пользователю (если скрыто)
    app7.HideMessage = const7.ksHideMessageNo       # Отвечаем НЕТ на любые вопросы программы

    table = []                                      # Создаём таблицу параметров
    for path in paths:
        doc7 = app7.Documents.Open(PathName=path,
                                   Visible=True,
                                   ReadOnly=True)       # Откроем файл в видимом режиме без права его изменять

        row = amount_sheet(doc7)                        # Посчитаем кол-во листов каждого формат
        row.update(stamp(doc7))                         # Читаем основную надпись
        row.update({
            "Filename": doc7.Name,                      # Имя файла
            "CountTD": count_demand(doc7, module7),     # Количество пунктов технических требований
            "CountDim": count_dimension(doc7, module7), # Количество размеров на чертеже
        })
        table.append(row)                               # Добавляем строку параметров в таблицу

        doc7.Close(const7.kdDoNotSaveChanges)           # Закроем файл без изменения

    if not is_run: app7.Quit()                          # Выходим из программы
    return table
```

## Диалоговое окно выбора файлов
Для удобного использования нашего скрипта воспользуемся возможностями стандартного модуля tkinter и выведем диалоговое окно выбора файлов:

```
from tkinter import Tk
from tkinter.filedialog import askopenfilenames

if __name__ == "__main__":
    root = Tk()
    root.withdraw()  # Скрываем основное окно и сразу окно выбора файлов

    filenames = askopenfilenames(title="Выберете чертежи деталей",
                                 filetypes=[('Kompas 3D', '*.cdw'),])

    print_to_excel(parse_design_documents(filenames))

    root.destroy()  # Уничтожаем основное окно
    root.mainloop()
```

## Отчётность
Чтобы не рисовать в tkintere интерфейс пользователя, предлагаю воспользоваться хорошей программой Excel, куда и выведем результат нашего труда:

```
def print_to_excel(result):
    excel = Dispatch("Excel.Application")  # Подключаемся к Excel
    excel.Visible = True                   # Делаем окно видимым
    wb = excel.Workbooks.Add()             # Добавляем новую книгу
    sheet = wb.ActiveSheet                 # Получаем ссылку на активный лист

    # Создаём заголовок таблицы
    sheet.Range("A1:J1").value = ["Имя файла", "Разработчик",
                                  "Кол-во размеров", "Кол-во пунктов ТТ",
                                  "А0", "А1", "А2", "А3", "А4", "Масштаб"]

    # Заполняем таблицу
    for i, row in enumerate(result):
        sheet.Range("A%d:J%d" % i+2).value = [row['Filename'],
                                              row['Designer'],
                                              row['CountDim'],
                                              row['CountTD'],
                                              row['A0'],
                                              row['A1'],
                                              row['A2'],
                                              row['A3'],
                                              row['A4'],
                                              "".join(('="', row['Scale'], '"'))]
```

Если не полениться и правильно подготовить Excel файл, то результат работы нашего скрипта можно сразу представить в наглядном виде:

![Здесь красивая цветная картинка из Excel](https://habrastorage.org/files/368/ebd/e0b/368ebde0bc76418b91e57ab9e5a1e7b6.png)

На графике изображено участие каждого сотрудника отдела в выпуске документации на изделие.

## Заключение
Используя скрипт для чертежа, созданного специально для данной статьи, мы получим следующие результаты: 
   * Количество размеров: 25 штук
   * Количество пуктов технических требований: 3 штуки
   * Масштаб: 1:1
   * Количетво листов: 1, формата А3
   
Трудозатраты, согласно упомянатому в начале статьи документу, составили: 1 час 20 минут. Как ни странно, примерно столько и было потрачено времени на разработку чертежа. 
Конечно, для внедрения подобных норм на предприятии нужны более серьёзные исследования и совершенно другие объёмы конструкторской документации. Данная же статья поможет упростить работу с API Kompas 3D при решении аналогичных задач.
Буду рад любым вашим замечаниями и предложениями к статье.

## Исходный код
* [GitHub](https://github.com/Weltraum/Article-Python-in-aid-of-the-designer.-Tames-API-Kompas-3D.)

## Ресурсы в помощь:
1. SDK (<i>C:\Program Files\ASCON\KOMPAS-3D V16\SDK\SDK.chm</i>)
2. [forum.ascon.ru](forum.ascon.ru)
