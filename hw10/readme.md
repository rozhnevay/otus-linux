# **Homework 10**

Реализовать 2 конкурирующих процесса по CPU. пробовать запустить с разными nice
### **Подготовим скрипт для расчета факториала большого числа**
```bash
    cat <<-"EOF" > fact.py
import time
import sys

start_time = time.time()

n = 233300
fact = 1

for i in range(1,n+1):
    fact = fact * i
print("Nice = %s seconds = %d" %(sys.argv[1], time.time() - start_time))
EOF
```

### **Подготовим скрипт-обертку**
```bash
    cat <<-"EOF" > fact-wrapper.sh
#!/usr/bin/env bash

NICE_1=-1
NICE_2=-30

nice -${NICE_1} python fact.py ${NICE_1} &
nice -${NICE_2} python fact.py ${NICE_2} &

exit 0
EOF
```
### **Дадим права на запуск**
```
    chmod o+x fact-wrapper.sh
```

### **Такую картину видим в TOP:**
```
PID USER      PR  NI    VIRT    RES    SHR S %CPU %MEM     TIME+ COMMAND        
23262 root       0 -20  133424  12364   2032 R 97.7  1.2   0:07.38 python         
23261 root      19  -1  132896  11960   2032 R  1.3  1.2   0:00.09 python       
```
### **Результат**
```
    Nice = -30 seconds = 34
    Nice = -1 seconds = 68
```

Вывод - nice работает)