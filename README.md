# Анализатор активности вопросов

## Цель проекта

Этот проект подключается к базе данных MySQL, извлекает данные о вопросах и предоставляет визуализации активности вопросов во времени. Он генерирует аналитические данные об объеме вопросов по месяцам, распределении вопросов по времени суток и распределении вопросов по дням недели.

<details>
  <summary>Технологии</summary>

  - **Python:** Основной язык программирования.
  - **Pandas:** Манипулирование и анализ данных.
  - **pymysql:** Коннектор MySQL.
  - **matplotlib:** Визуализация данных.
  - **seaborn:** Улучшенная визуализация данных.
  - **SQLAlchemy:** Инструментарий для взаимодействия с базами данных.
  - **numpy:** Библиотека для численных вычислений, используемая для сглаживания.
</details>

## Этапы проекта

1.  **Подключение к базе данных:** Подключается к базе данных MySQL с использованием SQLAlchemy.
2.  **Извлечение данных:** Извлекает данные `id` вопроса и `created_at` из таблицы `question_answers`.
3.  **Преобразование данных:** Преобразует столбец `created_at` в формат datetime.
4.  **Агрегация данных:**
    *   Вычисляет общее количество вопросов.
    *   Передискретизирует данные для подсчета вопросов по месяцам.
    *   Группирует данные по часам дня для подсчета вопросов по часам.
    *   Группирует данные по дням недели для подсчета вопросов по дням.
5.  **Визуализация данных:** Генерирует три графика:
    *   Линейный график, показывающий количество вопросов по месяцам.
    *   Линейный график, показывающий распределение вопросов по времени суток (сглаженный с использованием скользящего среднего).
    *   Горизонтальная столбчатая диаграмма, показывающая распределение вопросов по дням недели.



## 1.Настройки подключения к MySQL

```python

import matplotlib
 
matplotlib.use('TkAgg')
 
import pymysql
 
import pandas as pd
 
import matplotlib.pyplot as plt
 
import seaborn as sns
 
from sqlalchemy import create_engine
 
from wordcloud import WordCloud  # For word cloud
 
from collections import Counter  # For counting words
 
import numpy as np  # For smoothing
 
 
host = '********'
 
user = '****'
 
password = '****'
 
database = 'ppokob2_ppokob2'
 
port = 3306
 

engine = None
 
try:
 
    engine = create_engine(f'mysql+pymysql://{user}:{password}@{host}:{port}/{database}')
 
    print("Успешное создание SQLAlchemy Engine!")
 
except Exception as e:
 
    print(f"Ошибка при создании SQLAlchemy Engine: {e}")
 
    engine = None
```
  
## 2.Извлечение данных

```python 

if engine is not None:
 
    try:
 
        # Извлечение данных из таблицы
 
        sql_query = """
 
        SELECT 
 
            id,
 
            question,
 
            created_at
 
        FROM question_answers
 
        """  
 
        try:  # Вложенный try для обработки ошибок при запросе к БД
 
            df = pd.read\_sql(sql\_query, engine)  # Используем pd.read_sql для загрузки в DataFrame
 
            print("Данные успешно загружены из MySQL!")
 
        except Exception as e:
 
            print(f"Ошибка при загрузке данных из MySQL: {e}")
 
            df = None  # Устанавливаем df в None, если произошла ошибка
 
  ```
# Визуализация
<details>
  <summary>1.**Объем вопросов по месяцам:**</summary>
 
          -   Передискретизирует данные для подсчета вопросов по месяцам.
          -   Создает линейный график, показывающий тенденцию объема вопросов во времени.
          -   Аннотирует график общим количеством вопросов.
```python 
 
        if df is not None:
 
            try:
 
                # Преобразование created_at в datetime
 
                df\['created\_at'\] = pd.to\_datetime(df\['created_at'\], errors='coerce')
 
  
 
                # Calculate total questions
 
                total_questions = len(df)
 
  
 
                # Resample to get monthly counts
 
                monthly\_counts = df.set\_index('created_at')\['id'\].resample('ME').count()
 
  
 
                # Create the plot
 
                plt.figure(figsize=(12, 6))
 
                ax = monthly_counts.plot(
 
                    color='#ff7f0e',  # Bright orange color
 
                    linewidth=2,
 
                    marker='o',
 
                    markersize=5
 
                )
 
 
                # Add annotation for total questions
 
                ax.annotate(f'Всего: {total_questions}',
 
                            xy=(0.8, 0.9),  # Coordinates for the annotation
 
                            xycoords='axes fraction',  # Use axes coordinates
 
                            fontsize=12,
 
                            color='black')
 
                plt.title('Количество вопросов по месяцам', fontsize=14)
 
                plt.xlabel('Месяц', fontsize=12)
 
                plt.ylabel('Количество вопросов', fontsize=12)
 
                plt.grid(True)
 
                plt.show()
 
  ```
</details>
  2.**Распределение вопросов по времени суток:**
          -   Извлекает час дня из столбца `created_at`.
          -   Группирует данные по часам для подсчета вопросов по часам.
          -   Применяет метод сглаживания скользящим средним к почасовым данным.
          -   Определяет пиковые часы активности вопросов.
          -   Создает линейный график, показывающий распределение вопросов по времени суток.
          -   Аннотирует график количеством вопросов в каждый пиковый час.
  
```python        
 
                plt.figure(figsize=(14, 7))
 
                df\['hour'\] = df\['created_at'\].dt.hour
 
                hourly_counts = df.groupby('hour')\['id'\].count()
 
  
 
                # Сглаживание графика (скользящее среднее)
 
                window_size = 3
 
                hourly\_counts\_smooth = hourly\_counts.rolling(window=window\_size, center=True).mean()
 
                hourly\_counts\_smooth = hourly\_counts\_smooth.bfill().ffill()  # Заполнение NaN
 
  
 
                # Поиск пиков
 
                peak\_hours = hourly\_counts\_smooth\[hourly\_counts\_smooth == hourly\_counts_smooth.max()\].index.tolist()
 
  
 
                # Plotting
 
                plt.plot(hourly\_counts\_smooth.index, hourly\_counts\_smooth.values, color='#66b3ff', linewidth=2.5,
 
                         marker='o', markersize=8, markeredgecolor='white')
 
 
                # Добавление аннотаций для пиков
 
                for hour in peak_hours:
 
                    plt.annotate(f'{hourly\_counts\[hour\]}', xy=(hour, hourly\_counts_smooth\[hour\]),
 
                                 xytext=(hour, hourly\_counts\_smooth\[hour\] + 5),
 
                                 ha='center', fontsize=10, color='#333333',
 
                                 arrowprops=dict(facecolor='black', shrink=0.05))
 
                plt.title('Распределение вопросов по времени суток', fontsize=16, color='#333333')
 
                plt.xlabel('Час', fontsize=12, color='#555555')
 
                plt.ylabel('Количество вопросов', fontsize=12, color='#555555')
 
                plt.xticks(range(24), fontsize=10, color='#555555')
 
                plt.yticks(fontsize=10, color='#555555')
 
                # Убираем рамку графика
 
                sns.despine()
 
                # Убираем сетку
 
                plt.grid(False)
 
                # Отображаем график
 
                plt.show()
```
3.**Распределение вопросов по дням недели:**
          -   Извлекает день недели из столбца `created_at`.
          -   Подсчитывает количество вопросов, заданных в каждый день недели.
          -   Создает горизонтальную столбчатую диаграмму, показывающую распределение вопросов по дням недели, используя цветовую палитру "viridis" из Seaborn.
               
```python  
                plt.figure(figsize=(12, 6))
 
                df\['day\_of\_week'\] = df\['created\_at'\].dt.day\_name()
 
                day_order = \['Monday', 'Tuesday', 'Wednesday', 'Thursday', 'Friday', 'Saturday', 'Sunday'\]
 
                day\_counts = df\['day\_of\_week'\].value\_counts().reindex(day_order)
 
 
                # Choose a color palette
 
                palette = sns.color\_palette("viridis", len(day\_counts))  # Use viridis palette
 
  
                day_counts.plot(kind='barh', color=palette)  # Apply palette
 
                plt.title('Распределение вопросов по дням недели', fontsize=14)
 
                plt.xlabel('Количество вопросов', fontsize=12)
 
                plt.ylabel('День недели', fontsize=12)
 
                plt.show()
 
                print("Визуализация завершена.")
 
            except AttributeError as e:
 
                print(
 
                    f"Ошибка при обработке дат: {e}. Убедитесь, что столбец created_at существует и содержит корректные данные.")
 
            except Exception as e:
 
                print(f"Ошибка при визуализации: {e}")
 
    except Exception as e:
 
        print(f"Общая ошибка при загрузке данных и анализе: {e}")
```
