# Scripts
Scripts for GNU/Linux arch on phyton
#!/bin/bash

# Интервал проверки состояния системы (в секундах)
CHECK_INTERVAL=10

# Пороговые значения (настраиваемые)
CPU_THRESHOLD=80  # Максимальная загрузка CPU в процентах
MEM_THRESHOLD=80  # Максимальное использование памяти в процентах

# Функция для изменения приоритета процесса
adjust_priority() {
    local pid=$1
    local current_priority=$2
    local new_priority=$3

    if [[ $current_priority -ne $new_priority ]]; then
        echo "Изменение приоритета процесса PID=$pid с $current_priority на $new_priority"
        renice $new_priority -p $pid >/dev/null 2>&1
    fi
}

# Основной цикл
while true; do
    echo "Проверка состояния системы..."

    # Получение использования CPU и памяти
    cpu_usage=$(top -bn1 | grep "Cpu(s)" | awk '{print $2 + $4}')
    mem_usage=$(free | grep Mem | awk '{print $3/$2 * 100.0}')

    echo "Использование CPU: $cpu_usage%"
    echo "Использование памяти: $mem_usage%"

    # Если загрузка CPU или памяти превышает порог, изменяем приоритеты процессов
    if (( $(echo "$cpu_usage > $CPU_THRESHOLD" | bc -l) )) || (( $(echo "$mem_usage > $MEM_THRESHOLD" | bc -l) )); then
        echo "Высокая загрузка ресурсов, изменение приоритетов процессов..."

        # Получение списка процессов с наибольшей загрузкой CPU
        ps -eo pid,%cpu,%mem,pri,comm --sort=-%cpu | head -n 10 | while read pid cpu mem pri comm; do
            if [[ $pid =~ ^[0-9]+$ ]] && (( $(echo "$cpu > 10" | bc -l) )); then
                new_priority=10  # Новый приоритет, например, 10
                adjust_priority $pid $pri $new_priority
            fi
        done
    else
        echo "Ресурсы в норме."
    fi

    # Ожидание перед следующим циклом проверки
    sleep $CHECK_INTERVAL
done
