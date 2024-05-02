# Тестирование с использованием Helgrind и Thread Sanitizer(TSAN)

Для тестирования был выбран проект с одной из возможных реализаций задачи об обедающих философах ([
_ссылка_](https://github.com/Hizenberg469/Dining_philosopher/commit/92de58bdc273c8102c3e42f177353b298d7ddd51)).
В первую очередь проект был взят из-за своей относительной простоты.

## Thread sanitizer (TSAN)

> Компиляция с использованием `thread sanitizer` осуществлялась через `CMake`:
>```cmake
> set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fsanitize=thread")
>```

Запуск на проекте без изменений (искусственная гонка данных не добавлялась) дал следующие результаты.

`Thread sanitizer` не нашел как таковых ошибок, однако вывел предупреждение:

```bash
WARNING: ThreadSanitizer: lock-order-inversion (potential deadlock) (pid=228897)
  Cycle in lock order graph: M0 (0x5574c075e1d8) => M1 (0x5574c075e240) => M0
```

В нем говориться о наличии цикла в графе порядка взятия lock-ов. Это означает, что поток захватывает Mutex M1,
удерживая Mutex M0, а затем также захватывает Mutex M0, удерживая Mutex M1, что создает потенциальную ситуацию взаимной
блокировки. Однако на деле такой проблемы не возникает, потому что Mutex-ы расположены циклически.
> Другими словами, в задаче об обедающих философах нет двух философов,
> для которых _вилка M0_ является одновременно правой (для обоих), а _вилка M1_ является левой (для обоих).


Было решено проверить эффективность работы `Thread sanitizer` через добавление искусственной гонки данных (которой не
было в изначальном коде проекта).

Был убран один из Mutex-ов, отвечающих за блокировку _вилки_(`spoon_t`).
> Т.е. изменение общих данных происходило без примитивов синхронизации.

В результате `thread sanitizer` успешно нашел гонку данных и вывел следующее предупреждение, указав на проблемные места
в коде:

```bash
WARNING: ThreadSanitizer: data race (pid=231908)
  Write of size 1 at 0x565065dd129c by thread T5 (mutexes: write M0):
    #0 philosopher_access_both_spoons ./DiningPhilosopher.c:151 (DiningPhilosopher+0x1bbf) (BuildId: ...)
    #1 philosopher_fn ./DiningPhilosopher.c:197 (DiningPhilosopher+0x1e6a) (BuildId: ...)

  Previous write of size 1 at 0x565065dd129c by thread T4 (mutexes: write M1):
    #0 philosopher_access_both_spoons ./DiningPhilosopher.c:167 (DiningPhilosopher+0x1d3c) (BuildId: ...)
    #1 philosopher_fn ./DiningPhilosopher.c:197 (DiningPhilosopher+0x1e6a) (BuildId: ...)

  Location is global 'spoon' of size 520 at 0x565065dd1160 (DiningPhilosopher+0x529c)
```

## Helgrind

> Запуск `helgrind` осуществлялся следующим образом:
>```bash
> valgrind --tool=helgrind ./DiningPhilosopher
>```

Запуск на неизмененном проекте выявил наличие гонки данных:

```bash 
==240070== Possible data race during write of size 1 at 0x10C0E4 by thread #2
==240070== Locks held: 1, at address 0x10C0F0
==240070==    at 0x10951B: philosopher_release_both_spoons (DiningPhilosopher.c:74)
==240070==    by 0x1099B3: philosopher_fn (DiningPhilosopher.c:202)
==240070==
==240070== This conflicts with a previous write of size 1 by thread #3
==240070== Locks held: none
==240070==    at 0x109593: philosopher_release_both_spoons (DiningPhilosopher.c:86)
==240070==    by 0x1099B3: philosopher_fn (DiningPhilosopher.c:202)
==240070==  Address 0x10c0e4 is 4 bytes inside data symbol "spoon"
```

Однако на деле исполняемый код в этих местах написан с использованием Mutex-ов, поэтому гонки данных там (вроде) быть не
должно.
> В любом случае происходит перезапись одних и тех же значений, поэтому даже при условии гонки корректность программы не
> измениться.

Далее была внесена такая же гонка данных, как и в случае с `thread sanitizer`. `Helgrind` успешно обнаружил ее и вывел
подробные логи (`thread sanitizer` указал на меньшее количество проблемных мест, которые породил убранный лок):
> Пример одного из таких логов приведен ниже

```bash
==242336== Possible data race during read of size 1 at 0x10C104 by thread #3
==242336== Locks held: 1, at address 0x10C110
==242336==    at 0x1097B9: philosopher_access_both_spoons (DiningPhilosopher.c:151)
==242336==    by 0x109977: philosopher_fn (DiningPhilosopher.c:199)
==242336==
==242336== This conflicts with a previous write of size 1 by thread #2
==242336== Locks held: 1, at address 0x10C1E0
==242336==    at 0x1098AC: philosopher_access_both_spoons (DiningPhilosopher.c:168)
==242336==    by 0x109977: philosopher_fn (DiningPhilosopher.c:199)
==242336==  Address 0x10c104 is 4 bytes inside data symbol "spoon"
```

## Вывод

Можно сказать, что `thread sanitizer` и `helgrind` взаимозаменяемые инструменты. Однако, как показывает проведенные "
эксперимент", `helgrind` выводит немного больше информации о потенциальных ошибках.
