# homework14
#Настройка мониторинга. Prometheus
```
1. Обновляем ВМ
    yum update -y
2. Добавляем dns google
    echo "nameserver 8.8.8.8" >> /etc/resolv.conf
3. Скачиваем и устанавливаем Prometheus
    wget https://github.com/prometheus/prometheus/releases/download/v2.44.0/prometheus-2.44.0.linux-amd64.tar.gz
4. Создаем пользователя и каталоги, настраиваем для них владельцев
    useradd --no-create-home --shell /bin/false prometheus
    mkdir /etc/prometheus
    mkdir /var/lib/prometheus
    chown prometheus:prometheus /etc/prometheus
    chown prometheus:prometheus /var/lib/prometheus
5. Распаковываем архив, для удобства переименовываем директорию и копируем бинарники в /usr/local/bin
    tar -xvzf prometheus-2.44.0.linux-amd64.tar.gz
    mv prometheus-2.44.0.linux-amd64 prometheuspackage
    cp prometheuspackage/prometheus /usr/local/bin/
    cp prometheuspackage/promtool /usr/local/bin/
6. Меняем владельцев у бинарников
chown prometheus:prometheus /usr/local/bin/prometheus
chown prometheus:prometheus /usr/local/bin/promtool
7. По аналогии копируем библиотеки
    cp -r prometheuspackage/consoles /etc/prometheus
    cp -r prometheuspackage/console_libraries /etc/prometheus
    chown -R prometheus:prometheus /etc/prometheus/consoles
    chown -R prometheus:prometheus /etc/prometheus/console_libraries
8. Создаем конфигурационный файл prometheus.yml
    touch /etc/prometheus/prometheus.yml
    cat <<'EOF' >> /etc/prometheus/prometheus.yml
    ---
    global:
      scrape_interval: 10s

    scrape_configs:
      - job_name: 'prometheus_master'
        scrape_interval: 5s
        static_configs:
          - targets: ['localhost:9090']
    ...
    EOF
    chown prometheus:prometheus /etc/prometheus/prometheus.yml
9. Настраиваем сервис prometheus.service
    touch /etc/systemd/system/prometheus.service
    cat <<'EOF'>> /etc/systemd/system/prometheus.service
    [Unit]
    Description=Prometheus
    Wants=network-online.target
    After=network-online.target

    [Service]
    User=prometheus
    Group=prometheus
    Type=simple
    ExecStart=/usr/local/bin/prometheus \
    --config.file /etc/prometheus/prometheus.yml \
    --storage.tsdb.path /var/lib/prometheus/ \
    --web.console.templates=/etc/prometheus/consoles \
    --web.console.libraries=/etc/prometheus/console_libraries

    [Install]
    WantedBy=multi-user.target
    EOF
10. Добавляем правило в firewall
     firewall-cmd --zone=public --add-port=9090/tcp --permanent
     firewall-cmd --zone=public --add-port=9100/tcp --permanent
     firewall-cmd --zone=public --add-port=9094/tcp --permanent
     firewall-cmd --zone=public --add-port=9094/udp --permanent
     firewall-cmd --zone=public --add-port=9094/udp --permanent
     firewall-cmd --zone=public --add-port=3000/tcp --permanent
     systemctl reload firewalld
11. Переопрашиваем сервисы и запускаем Prometheus
     systemctl daemon-reload
     systemctl start prometheus
     systemctl status prometheus
12. Скачиваем и распаковываем Node Exporter
     wget https://github.com/prometheus/node_exporter/releases/download/v1.5.0/node_exporter-1.5.0.linux-amd64.tar.gz
     tar xzfv node_exporter-1.5.0.linux-amd64.tar.gz
13. Создаем пользователя, перемещаем бинарник в /usr/local/bin
     useradd -rs /bin/false nodeusr
     mv node_exporter-1.5.0.linux-amd64/node_exporter /usr/local/bin/
14. Создаем сервис node_exporter.service
     touch /etc/systemd/system/node_exporter.service
     cat <<'EOF'>> /etc/systemd/system/node_exporter.service
     [Unit]
     Description=Node Exporter
     After=network.target

     [Service]
     User=nodeusr
     Group=nodeusr
     Type=simple
     ExecStart=/usr/local/bin/node_exporter

     [Install]
     WantedBy=multi-user.target
     EOF
15. Запускаем сервис
     systemctl daemon-reload
     systemctl start node_exporter
     systemctl enable node_exporter
16. Обновляем конфигурацию Prometheus
     sed -i d /etc/prometheus/prometheus.yml
     cat <<'EOF'>> /etc/prometheus/prometheus.yml
     ---
     global:
       scrape_interval: 10s

     scrape_configs:
       - job_name: 'prometheus_master'
         scrape_interval: 5s
         static_configs:
           - targets: ['localhost:9090']
       - job_name: 'node_exporter_centos'
         scrape_interval: 5s
         static_configs:
           - targets: ['localhost:9100', '192.168.1.128:9100']     
     ...
     EOF
17. Перезапускаем сервис
     systemctl restart prometheus
18. Устанавливаем Grafana
     yum -y install /vagrant/grafana_enterprise_9.5.2_1.x86_64-364648-5d442e.rpm
19. Стартуем сервис
     systemctl daemon-reload
     systemctl start grafana-server
20. Скачиваем и распаковываем AlertManager
     wget https://github.com/prometheus/alertmanager/releases/download/v0.25.0/alertmanager-0.25.0.linux-amd64.tar.gz
     tar zxf alertmanager-0.25.0.linux-amd64.tar.gz
21. Создаем пользователя и директории"
     useradd --no-create-home --shell /bin/false alertmanager
     usermod --home /var/lib/alertmanager alertmanager
     mkdir /etc/alertmanager
     mkdir /var/lib/prometheus/alertmanager
22. Копируем бинарники из архива в /usr/local/bin и меняем владельца
     cp alertmanager-0.25.0.linux-amd64/amtool /usr/local/bin/
     cp alertmanager-0.25.0.linux-amd64/alertmanager /usr/local/bin/
     cp alertmanager-0.25.0.linux-amd64/alertmanager.yml /etc/alertmanager/
     chown -R alertmanager:alertmanager /etc/alertmanager /var/lib/prometheus/alertmanager
     chown alertmanager:alertmanager /usr/local/bin/{alertmanager,amtool}
     echo "ALERTMANAGER_OPTS=\"\"" > /etc/default/alertmanager
     chown alertmanager:alertmanager /etc/default/alertmanager
     chown -R alertmanager:alertmanager /var/lib/prometheus/alertmanager
23. Настраиваем сервис
     touch /etc/systemd/system/alertmanager.service
     cat <<'EOF'>> /etc/systemd/system/alertmanager.service
     [Unit]
     Description=Alertmanager Service
     After=network.target prometheus.service

     [Service]
     EnvironmentFile=-/etc/default/alertmanager
     User=alertmanager
     Group=alertmanager
     Type=simple
     ExecStart=/usr/local/bin/alertmanager \
     --config.file=/etc/alertmanager/alertmanager.yml \
     --storage.path=/var/lib/prometheus/alertmanager \
     $ALERTMANAGER_OPTS
     ExecReload=/bin/kill -HUP $MAINPID
     Restart=on-failure
     Restart=always

     [Install]
     WantedBy=multi-user.target
     EOF
24. Запускаем сервис
     systemctl daemon-reload
     systemctl start alertmanager
25. Настраиваем правила
     touch /etc/prometheus/rules.yml
     cat <<'EOF'>> /etc/prometheus/rules.yml
     groups:
       - name: alert.rules
         rules:
         - alert: InstanceDown
           expr: up == 0
           for: 1m
           labels:
             severity: critical
           annotations:
             description: '{{ $labels.instance }} of job {{ $labels.job }} has been down for more than 1 minute.'
             summary: Instance {{ $labels.instance }} down
     EOF
26. Проверяем валидность
     /usr/local/bin/promtool check rules /etc/prometheus/rules.yml
27. Обновляем конфигурацию Prometheus
     sed -i d /etc/prometheus/prometheus.yml
     cat <<'EOF'>> /etc/prometheus/prometheus.yml
     ---
     global:
       scrape_interval: 10s

     scrape_configs:
       - job_name: 'prometheus_master'
         scrape_interval: 5s
         static_configs:
           - targets: ['localhost:9090']
       - job_name: 'node_exporter_centos'
         scrape_interval: 5s
         static_configs:
           - targets: ['localhost:9100', '192.168.1.128:9100']
           
     rule_files:
       - "rules.yml"
     alerting:
       alertmanagers:
       - static_configs:
         - targets:
           - localhost:9093     
     ...
     EOF
28. Рестартуем сервисы
     systemctl restart prometheus
     systemctl restart alertmanager
29. Запускаем на host браузер с адресом http://127.0.0.1:9090
    В Prometeus  добавляем панели для мониторинга локального компьютера и удаленного:
    - свободное место на дисках: node_filesystem_free_bytes{fstype=~"ext4|xfs"} / node_filesystem_size_bytes{fstype=~"ext4|xfs"}
    - объем принятых данных по сети: rate(node_network_receive_bytes_total[5m]) * 8 / 1024 / 1024
    - усредненная свободная память: 100 * (1 - ((avg_over_time(node_memory_MemFree_bytes[5m]) + avg_over_time(node_memory_Cached_bytes[5m]) + avg_over_time(node_memory_Buffers_bytes[5m])) / avg_over_time(node_memory_MemTotal_bytes[5m])))
    - Среднее значение ожидания ЦПУ за 5 минут для всех узлов: avg(irate(node_cpu_seconds_total{mode="iowait"}[5m])) * 100
30. В Grafana также добавляем панели с теми же  запросами.

```
