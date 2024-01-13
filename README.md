# Задача
## Приоритетный доступ к GPU

В системе имеется N обычных процессов и 1 приоритетный процесс, желающих загрузить свою задачу в GPU. 
Процесс, Отправивший задача на GPU, занимает его от 1 до 4 тактов. Обычные процессы равноправны, 
сначала должен получить обслуживание процесс, выставивший заявку раньше. Приоритетный процесс получает
обслуживание вне очереди, сразу, как освободится  GPU. Необходимо реализовать процедуру доступа к GPU. 
После завершения выполнения заявки процесс не может мгновенно выставить следующую.

## Проверяемые ключевые свойства системы:
1. Эксклюзивность получения обслуживания на GPU - только один процесс может обслуживаться в один момент времени
2. Обычный процесс, выставивший заявку раньше, получает заявку раньше
3. Приоритетный процесс получает обслуживание при первой возможности
4. Процесс, выставивший заявку, обязательно получает обслуживание. 

В ходе работы следует “испортить” алгоритм, когда свойства перестают выполняться.

# Решение

Основная деталь реализации — использование нескольких объектов типа `task`, каждый из которых представляет
определенную позицию каждой из задач в очереди. Например, `task0` показывает описание задачи в голове очереди,
`task1` — следующей за ней задачи и т.д. При загрузке задачи в GPU ее свойства копируются в свойство 
`currentTask`. Задача с высоким приоритетом не хранится в очереди вообще. Вместо этого для её хранения
есть специальные свойства `highPriorityTask` и `isHighPriorityTaskInQueue`. Первое показывает свойства задачи,
второе — флаг, обозначающий, что один из процессов породил задачу с высоким приоритетом и её необходимо загрузить
в GPU сразу же после завершения исполнения текущей задачи.

## Проверка свойств

1. Эксклюзивность получения обслуживания на GPU - только один процесс может обслуживаться в один момент времени

    По построению системы, в один момент времени в GPU может быть только одна задача (так как нет других свойств
    для хранения задач в GPU)

2. Обычный процесс, выставивший заявку раньше, получает заявку раньше

    Свойство выполняется из-за того, что очередь работает корректно. Можно нарушить порядок перемещения элементов
    в ней, чтобы свойство перестало выполняться (`solution_violate2.smv`)

3. Приоритетный процесс получает обслуживание при первой возможности

    Свойство выполняется засчет дополнительных условий при выборе задачи на загрузку в GPU. Если их изменить, то 
    свойства перестанут выполняться (`solution_violation3.smv`)

4. Процесс, выставивший заявку, обязательно получает обслуживание. 

    Свойство выполняется по причине корректности очереди (если задача попала в очередь, то она выйдет из нее рано 
    или поздно), по причине корректности кода выбора задачи для загрузки в GPU и из-за того, что приоритетная
    задача не может появиться сразу же после завершения предыдущей приоритетной. При изменении последнего свойства
    (`solution_violation4.svm`) свойство перестанет выполняться

При верификации спецификации файла `solution.smv` получается следующий вывод:
```
nuXmv > reset
nuXmv > read_model -i /Users/vfeofilaktov/labs/verification-lab4-nuxmv/solution.smv
nuXmv > go
nuXmv > check_ltlspec
-- specification  G (c.isHighPriorityTaskInQueue = TRUE ->  F c.gpuInst.currentTask.isHighPriority = TRUE)  is true
-- specification  G (((((((c.isHighPriorityTaskInQueue = TRUE & c.gpuInst.currentTask.isHighPriority = FALSE) & c.task0.processId = 0) & c.task1.processId = 1) & c.task2.processId = 2) & c.task3.processId = 3) & c.task4.processId = 4) ->  F (((((c.gpuInst.currentTask.isHighPriority = TRUE & c.task0.processId = 0) & c.task1.processId = 1) & c.task2.processId = 2) & c.task3.processId = 3) & c.task4.processId = 4))  is true
-- specification  G ((c.task0.processId = 0 & c.task1.processId = 1) ->  F ((c.gpuInst.currentTask.processId = 0 & c.gpuInst.currentTask.isHighPriority = FALSE) & c.task0.processId = 1))  is true
-- specification  G ((c.task1.processId = 1 & c.task2.processId = 2) ->  F ((c.gpuInst.currentTask.processId = 1 & c.gpuInst.currentTask.isHighPriority = FALSE) & c.task0.processId = 2))  is true
-- specification  G ((c.task2.processId = 2 & c.task3.processId = 3) ->  F ((c.gpuInst.currentTask.processId = 2 & c.gpuInst.currentTask.isHighPriority = FALSE) & c.task0.processId = 3))  is true
-- specification  G ((c.task3.processId = 3 & c.task4.processId = 4) ->  F ((c.gpuInst.currentTask.processId = 3 & c.gpuInst.currentTask.isHighPriority = FALSE) & c.task0.processId = 4))  is true
-- specification  G (c.task0.processId = 0 ->  F (c.gpuInst.currentTask.processId = 0 & c.gpuInst.currentTask.isHighPriority = FALSE))  is true
-- specification  G (c.task1.processId = 1 ->  F (c.gpuInst.currentTask.processId = 1 & c.gpuInst.currentTask.isHighPriority = FALSE))  is true
-- specification  G (c.task2.processId = 2 ->  F (c.gpuInst.currentTask.processId = 2 & c.gpuInst.currentTask.isHighPriority = FALSE))  is true
-- specification  G (c.task3.processId = 3 ->  F (c.gpuInst.currentTask.processId = 3 & c.gpuInst.currentTask.isHighPriority = FALSE))  is true
-- specification  G (c.task4.processId = 4 ->  F (c.gpuInst.currentTask.processId = 4 & c.gpuInst.currentTask.isHighPriority = FALSE))  is true
```