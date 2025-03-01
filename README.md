# Решение соревнования JPX Tokyo Market Prediction

Репозиторий содержит решения практического задания курса Light Auto ML:
- 2 базовых решения с использованием LAMA с разными конфигурациями
- 1 оптимальное решение с кастомным пайплайном обработки временных рядов

[![Code Style: PEP8](https://img.shields.io/badge/code%20style-PEP8-brightgreen.svg)](https://www.python.org/dev/peps/pep-0008/)

## Описание соревнования
**Задача**: Прогнозирование ранговой позиции акций на Японской бирже  
**Тип задачи**: Регрессия с временными рядами  
**Целевая метрика**: Sharpe Ratio спредов доходности  
**Особенности**:
- Динамический набор из ~2000 акций
- Запрет на "look-ahead" в тестовой фазе
- Реальная проверка на рыночных данных после дедлайна

**Ссылка**: [JPX Tokyo Market Prediction](https://www.kaggle.com/competitions/jpx-tokyo-stock-exchange-prediction)

## Реализации

### 1. Базовые решения (LAMA)
#### Конфигурация 1: LightGBM + CatBoost
```python
TabularAutoML(
    task=Task('reg', loss='mae'),
    general_params={'use_algos': [['lgb', 'cb']]},
    reader_params={'cv': 3, 'random_state': 42}
)
```
**Особенности**:
- 3-х фолдовая кросс-валидация
- Автоматический подбор признаков
- Обработка временных рядов через группировку по SecurityCode

#### Конфигурация 2: Расширенный набор моделей
```python
TabularAutoML(
    general_params={
        'use_algos': [['lgb', 'cb', 'lgb_tuned']],
        'max_intersection_depth': 3
    },
    reader_params={'cv': 5, 'test_size': 0.1}
)
```
**Улучшения**:
- Добавлена тонкая настройка LightGBM
- Увеличение глубины анализа взаимодействий признаков
- 5-фолдовая валидация с холд-аут выборкой 10%

### 2. Оптимальное решение
**Архитектура пайплайна**:
```
Raw Data → Ценовая коррекция → Feature Engineering → Стратегическое ранжирование
```

## Результаты
| Решение               | Sharpe Ratio | Время обучения | Ключевые особенности |
|-----------------------|--------------|----------------|-----------------------|
| LAMA (Config 1)       | -0.005         | 4m 38s         | Базовый AutoML        |
| LAMA (Config 2)       | 0.032         | 1h 42m 24s      | Расширенный набор моделей |
| **Optimal Solution**  | **0.347**     | 59s         | Кастомная архитектура |

![{3DE03226-7B0C-4452-8D2B-BB09734F2209}](https://github.com/user-attachments/assets/19d19b3a-db9b-4d09-9f1b-d69ca9519698)
![{18A267CA-EB0B-4CBC-9C3A-9AD8700DFDC7}](https://github.com/user-attachments/assets/3f8c760c-ad60-4fba-aff7-8c3e59552269)
![{3D365752-0492-45A3-97FA-776C9EBF9624}](https://github.com/user-attachments/assets/5d427238-fbfb-4ffa-b5b1-b18f08707e5a)
![{CD02DCB0-D9A5-4958-8C04-263389E90E6B}](https://github.com/user-attachments/assets/955d98bd-ed0b-48fd-8c8c-1e7141008d29)


## Ключевые особенности реализаций

### LAMA Решения
1. **Предобработка данных**:
```python
def adjust_price(price):
    # Корректировка цен с учётом корпоративных действий
    df['CumulativeAdjustmentFactor'] = df['AdjustmentFactor'].cumprod()
    df['AdjustedClose'] = ... # расчет скорректированной цены
```

2. **Генерация признаков**:
- Скользящие средние (5, 10, 20 дней)
- Историческая волатильность
- Флаги дивидендных выплат

3. **Валидация**:
```python
ReaderParams = {
    'cv': TimeSeriesSeriesSplit(n_splits=3),
    'test_size': timedelta(days=30)
}
```

### Алгоритм оптимального решения:

**Суть подхода**:  
Комбинация двух простых эвристик:
1. **Мгновенная доходность** (return_1day) - процентное изменение скорректированной цены за 1 день
2. **Дивидендный флаг** - бинарный признак наличия ожидаемых дивидендов

**Ключевые шаги**:
1. Корректировка исторических цен с учётом корпоративных действий
2. Расчёт 1-дневной доходности для каждого инструмента
3. Приоритизация акций *без* дивидендов через искусственное занижение их ранга
4. Ранжирование акций по убыванию/возрастанию доходности в зависимости от версии модели

### Почему это сработало?
1. **Дивидендный эффект**:  
   Акции с предстоящими дивидендами исторически показывали просадку в целевом периоде (t+2 дня), что позволило избежать убытков через их исключение из топ-рангов.

2. **Инверсия сигнала**:  
   Экспериментально обнаруженная отрицательная корреляция между 1-дневной доходностью и последующей динамикой цен.

3. **Устойчивость к шуму**:  
   Минимизация сложных преобразований снизила риск переобучения в условиях нестабильного рынка.

**Результат**:  
Сочетание простоты и анализа особенностей целевой переменной позволило достичь Sharpe Ratio **0.347** при практически нулевых вычислительных затратах.
Безусловно, отчасти, данный результат - скорее удачное стечение набора входных данных, которые не позволили lama построить необходимые зависимости для получения оптимального результата.
Однако это же подтверждает факт, что, иногда, простые алгоритмические решения лучше громоздких моделей машинного обучения

_Примечание: можно подумать, что проблема в данном случае именно в предобработке входных данных, поскольку lama получала больше значений на вход, и как следствие страдала от шума. Однако тест lama с параметрами, аналогичными алгоритмическому решению показал результат 0.051 и не был включен в итоговую работу из-за его примитивности_

**Требования**:
- Python 3.7+
- Особенности установки LAMA:
Из-за запрета на использование интернета при запуске, пришлось использовать локальную копию lama.
```bash
!pip install --no-index ../input/lightautoml/lightautoml-0.3.7.3-py3-none-any.whl
```

## Выводы
Как уже было сказано выше - основной вывод состоит в том, что, иногда, простые алгоритмические решения лучше громоздких моделей машинного обучения
