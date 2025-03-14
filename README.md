# k8s_falco

Этот проект представляет собой реализацию системы мониторинга безопасности контейнеров с использованием falco, настраивамом в kubernetes. При срабатывании системы мониторинга alert направляется в чат-бот telegram

# Запуск и настройка falco

1. Установка необходимого ПО (под ubuntu):
```./tools.sh```
2. Запуск k8s в minikube: ```minikube start --driver=docker ```
3. Создание необходимых namespace: ```kubectl create namespace secured-ns```
   ```kubectl create namespace dev```
   ```kubectl create namespace prod```
4. Создаем структуру для созданных namespace с помощью deployment.yaml: ```kubectl apply -f deployment.yaml -n secured-ns```
```kubectl apply -f deployment.yaml -n prod```
```kubectl apply -f deployment.yaml -n dev```
5. Добавляем репозиторий Faclo ```helm repo add falcosecurity https://falcosecurity.github.io/charts```
```helm repo update```
6. Устанавливаем FalcoSideKick: ```helm install falcosidekick -f values-kick.yaml falcosecurity/falcosidekick -n secured-ns```
7. Устанавливаем Falco ```helm install falco falcosecurity/falco -n secured-ns -f values-falco.yaml --set collectors.kubernetes.enabled=true --set-json 'falco.append_output=[{"match": {"source": "syscall"},"extra_output": "pod_uid=%k8smeta.pod.uid, pod_name=%k8smeta.pod.name, namespace_name=%k8smeta.ns.name"}]'``` Здесь помимо самой установки, устанавливаются параметры collectors.kubernetes для сбора метаданных, а также установка json структуры --set-json. Все это необходимо для дальнейшей настройки срабатываний, так как в обычной структуре поле k8s.ns.name будет пустым, мы создаем новое поле k8smeta.ns.name с помощью этих параметров, по которым в дальнейшем и будет происходить фильтрация (если не хочется подключать сюда metacollector, то можно убрать эти параметры и использовать фильтрацию по container.name через регулярные выражения)
* сейчас уже работает фильтрация, после указания api бота и chatid в values-kick.yaml дефолтные правила falco уже будут работать (лежат они в <FALCO_POD_NAME>:/etc/falco/falco_rules.yaml в secured-ns), так что получить alert в телеграм уже можно, просто войдя в под (kubectl exec -it <FALCO_POD_NAME> -n secured-ns -- /bin/bash)
# настройка дефолтных правил falco, а также создание собственного правила
* дефолтные правила будут срабатывать на действия, относящиеся к namespace prod
1. Копируем правила из falco пода к себе для удобного редактирования ```kubectl -n secured-ns exec -it <FALCO_POD_NAME> -- cat /etc/falco/falco_rules.yaml > combined_rules.yaml```
2. Изменяем правила путем добавления в каждое правило дополнительного условия ```k8smeta.ns.name = "prod"``` или (если metacollector не используется) ```container.name contains "_prod_"```
3. Копируем правила в falco под ```kubectl -n secured-ns cp combined_rules.yaml <FALCO_POD_NAME>:/etc/falco/falco_rules.yaml``` (в файлах лежит готовый combined_rules и combined_rules_with_meta с metacollector)
* на данный момент дефолтные правила уже работают только на события, касающиеся prod
4. Создаем свое правило. Дописываем в values-falco.yaml:
```
customRules:
  detect-cat-in-container.yaml: |-
      - rule: Terminal Shell with cat Command in Container
        desc: Detect when someone executes cat command inside a container
        condition: >
          evt.type = execve 
          and container 
          and proc.name = "cat"
          and container.name contains "_prod_" # without metacollector
          and k8smeta.ns.name = "prod" # with metacollector
        output: >
          CRITICAL: Cat command executed in a container 
          (user=%user.name uid=%user.uid process=%proc.name
          command=%proc.cmdline container_id=%container.id 
          container_name=%container.name k8s_ns=%k8s.ns.name)
        priority: CRITICAL
        tags: [security, terminal, exec]
```
Это правило срабатывает на операцию cat в prod, если изначально был взят весь файл values-falco.yaml, то правило уже записано и работает, иначе необходимо обновить pod: ```helm upgrade falco falcosecurity/falco -n secured-ns -f values-falco.yaml --set collectors.kubernetes.enabled=true --set-json 'falco.append_output=[{"match": {"source": "syscall"},"extra_output": "pod_uid=%k8smeta.pod.uid, pod_name=%k8smeta.pod.name, namespace_name=%k8smeta.ns.name"}]'```
(после обновления пода измененные дефолтные правила сбросятся, необходимо заново скопировать туда измененные см. шаг 3)
* сейчас все правила настроены и исправно работают
# Тестирование 3 дефолтных  и разработанного правила
1. alert на вход в систему: ```kubectl exec -it <ubuntu-POD_NAME> -n prod -- /bin/bash```
название правила: Terminal shell in container
2. alert на поиск чуствительных файлов: ```grep -r "PRIVATE KEY" /etc/```
название правила: Search for Sensitive Files
3. alert на создание сокета (сначала необходимо установить nmap ```apt update && apt install nmap -y```): ```nmap -p 80 example.com```
название правила: Packet socket created in container
4. alert на использование разработанного правила для cat: ```cat```
название правила: Terminal Shell with cat Command in Container
# Полезные команды
1. Вход в систему: ```kubectl exec -it <POD_NAME> -n <NAMESPACE> -- /bin/bash```
2. Вывод списка подов: ```kubectl get pods -n <NAMESPACE>```
3. Вывод списка namespace: ```kubectl get namespaces```
