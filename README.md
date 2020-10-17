swarm-crutchyorder
=====
Упоротый способ управлять очередностью загрузки Swarm-нод и системы хранения данных.

## Проблема
У нас есть несколько виртуальных машин — три Docker Swarm менеджера и одно хранилище.
Хранилище имеет несколько NFS-шар, используемых как персистеное хранилище для Volumes.

Но мы не можем гарантировать то, что Docker запустится на нодах только тогда, когда будет
готово хранилище, и это проблема — если очередность не сходится, то стэк падает и не поднимается.

## Решение
Ужасное. 

  1. Мы поднимаем на хранилище миниатюрный веб-сервер, но только тогда, когда NFS-сервер
     готов к работе (указываем nfs-server как dependency в systemd-службе).

  2. На хост-нодах Swarm, поднимаем службу, объявленную в зависимостях для Docker. 
     Внутри службы постоянно пытаемся достучаться до веб-сервера на хранилище, и при успешном
     ответе, запускают docker.service

## Установка
```bash
git clone https://github.com/b4ck5p4c3/swarm-crutchyorder
cd swarm-crutchyorder

# Adjust "Enviornment" parameters in .service files for both swarm and storage nodes
# (storage) BIND_PORT is TCP port to BIND by the heartbeat server
nano storage/docker-san-heartbeat.service

# (swarm) STORAGE_HEARTBEAT_URL is a URL to the heartbeat server
nano node/docker-wait-for-storage.service

# If it's a storage node
sudo cp storage/docker-san-heartbeat /usr/local/bin/
sudo cp storage/docker-san-heartbeat.service /etc/systemd/system/
sudo chmod +x /usr/local/bin/docker-san-heartbeat
sudo systemctl enable docker-san-heartbeat.service

# If it's a Swarm node
sudo cp node/docker-wait-for-storage /usr/local/bin/
sudo cp node/docker-wait-for-storage.service /usr/local/bin/
sudo chmod +x /usr/local/bin/docker-wait-for-storage
sudo systemctl enable docker-wait-for-storage.service
```
