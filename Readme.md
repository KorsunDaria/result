
# <span style="color: #1565c0;">Расширенное нагрузочное тестирование pgvector</span>

<div style="background-color: #e3f2fd; border-left: 5px solid #1e88e5; padding: 14px 18px; margin: 15px 0; border-radius: 0 6px 6px 0; font-size: 0.95em; color: #0d47a1;">
  <strong>О документе:</strong> продолжение оригинального отчёта по масштабированию HNSW — четыре направления, каждое закрывает свой открытый вопрос.
</div>

<hr style="border: none; height: 3px; background-color: #1e88e5; margin: 25px 0;" />

## <span style="color: #1565c0;">1. Направления исследования</span>

**1) Уровневый параллелизм при сборке** - 

**2) Рыночные расширения** - VectorChor (тестирование  RaBitQ) и pgvectorscale 

**3) RAM-порог через cgroups** - полная кривая деградации build time, что бы конкретезироаться необходимый запас ram 

**4) ef_search / recall при фильтрации** - разбор проблемы recall=0.012 на filter 99%, более 
подробноне тестирование разных сценариев и iterative_scan

<hr style="border: none; height: 3px; background-color: #1e88e5; margin: 25px 0;" />

## <span style="color: #1565c0;">2. Оборудование</span>

**Уже известно:**

- ноутбук, **32 ГБ RAM**
- ОС: Linux
- Postgres18_STABLE
- 

<hr style="border: none; height: 3px; background-color: #1e88e5; margin: 25px 0;" />

## <span style="color: #1565c0;">4. Направление 4. ef_search / recall при фильтрации</span>

**Цель:**

- в оригинале на 5M: recall = 0.012 (фильтр 99%)
- Преполагаемая версия маленький ef_search 

**Измеряли:**

1) recall и QPS по ef_search 32–512 (фильтр 99%, индекс 500K)
2) отдельно: режимы iterative_scan

![ef_search / recall](./report_summary.png)

**Итог** - кривая recall под фильтром 99% не зависит от ef_search — рост ef_search с 32 до 512 не улучшает recall, если iterative_scan = off. При этом abсолютные значения низкие: в среднем возвращается ~0.4 строки из 100 запрошенных, что подтверждает исходную проблему (recall = 0.012) на новом наборе данных.

Отдельно проверено: при высокой селективности фильтра планировщик Postgres предпочитает Bitmap Heap Scan вместо HNSW Index Scan даже при enable_seqscan=off (стоимостная модель считает его дешевле). Без явного enable_bitmapscan=off измерения recall фактически относятся не к HNSW-индексу, а к bitmap-скану.

**Вывод** - изначальная рекомендация "повышать ef_search" была актуальна до появления iterative_scan (pgvector 0.8.0+) и не решает проблему в принципе: ef_search управляет шириной поиска до фильтрации, а не количеством кандидатов, прошедших фильтр. Механизм iterative_scan=relaxed_order устраняет саму причину — граф дообходится до тех пор, пока не наберётся max_scan_tuples кандидатов, удовлетворяющих фильтру, вместо одного прохода top-ef_search.

При дефолтном max_scan_tuples = 20000 recall восстанавливается с ~0.4/100 до 100/100. При заниженном (max_scan_tuples = 2000) recall восстанавливается лишь частично (24.7/100) — то есть параметр не универсален и должен калиброваться под ожидаемую селективность фильтра.

**Итоговая рекомендация** - 
1) Использовать iterative_scan = relaxed_order как основной механизм борьбы с recall-деградацией под фильтром, а не увеличение ef_search.
2) Max_scan_tuples калибровать под целевую селективность фильтра (дефолт 20000 достаточен для 99%, но не универсален для более жёстких фильтров) — требует отдельного теста на кривой selectivity → optimal max_scan_tuples.

Пометка - смотеть что используется HNSW Index Scan или  Bitmap Heap Scan перед тем как менять параметры.
