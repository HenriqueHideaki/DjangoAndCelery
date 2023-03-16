# [Tarefas assíncronas com Django e Celery](https://realpython.com/asynchronous-tasks-with-django-and-celery/)

![Tela Inicial](source_code_initial/img/celery.png)


Você construiu um aplicativo Django brilhante e deseja lançá-lo ao público, mas está preocupado com as tarefas demoradas que fazem parte do fluxo de trabalho do seu aplicativo. Você não quer que seus usuários tenham uma experiência negativa ao navegar em seu aplicativo. Você pode integrar o Celery para ajudar nisso.

Celery é uma fila de tarefas distribuída para sistemas UNIX. Ele permite que você descarregue o trabalho de seu aplicativo Python. Depois de integrar o Celery ao seu aplicativo, você pode enviar tarefas demoradas para a fila de tarefas do Celery. Dessa forma, seu aplicativo da web pode continuar respondendo rapidamente aos usuários enquanto o Celery conclui operações caras de forma assíncrona em segundo plano.

Para receber tarefas de seu programa e enviar resultados para um back-end, o Celery requer um agente de mensagens para comunicação. Redis e RabbitMQ são dois corretores de mensagens que os desenvolvedores costumam usar junto com o Celery.

# Por que usar o Celery?
Existem duas razões principais pelas quais a maioria dos desenvolvedores deseja começar a usar o Celery:
*   Descarregar o trabalho do seu aplicativo para processos distribuídos que podem ser executados independentemente do seu aplicativo
* Agendar a execução de tarefas em um horário específico, às vezes como eventos recorrentes

O Celery é uma excelente escolha para esses dois casos de uso. Ele se define como “uma fila de tarefas com foco no processamento em tempo real, além de oferecer suporte ao agendamento de tarefas”

Embora essas duas funcionalidades façam parte do Celery, elas geralmente são abordadas separadamente:

* Celery workers: são processos de trabalho que executam tarefas independentemente umas das outras e fora do contexto de seu serviço principal.
* Celery beat: é um agendador que orquestra quando executar tarefas. Você também pode usá-lo para agendar tarefas periódicas.


## Como você pode aproveitar o Celery para seu aplicativo Django?

Celery não é útil apenas para aplicativos da Web, mas certamente é popular nesse contexto. Isso porque você pode enfrentar com eficiência algumas situações cotidianas no desenvolvimento da Web usando uma fila de tarefas distribuídas, como Celery:

* **Envio de e-mail**: você pode querer enviar uma verificação de e-mail, um e-mail de redefinição de senha ou uma confirmação de envio de formulário. O envio de e-mails pode demorar um pouco e deixar seu app lento, principalmente se ele tiver muitos usuários.

* **Processamento de imagem**: você pode querer redimensionar as imagens de avatar que os usuários carregam ou aplicar alguma codificação em todas as imagens que os usuários podem compartilhar em sua plataforma. O processamento de imagens geralmente é uma tarefa que consome muitos recursos e pode tornar seu aplicativo da web mais lento, principalmente se você estiver atendendo a uma grande comunidade de usuários.

* **Processamento de texto**: se você permitir que os usuários adicionem dados ao seu aplicativo, convém monitorar a entrada deles. Por exemplo, você pode querer verificar palavrões nos comentários ou traduzir o texto enviado pelo usuário para um idioma diferente. Lidar com todo esse trabalho no contexto de seu aplicativo da Web pode prejudicar significativamente o desempenho.

* **Chamadas de API e outras solicitações da Web**: se você precisar fazer solicitações da Web para fornecer o serviço que seu aplicativo oferece, poderá se deparar rapidamente com tempos de espera inesperados. Isso é válido tanto para solicitações de API com taxa limitada quanto para outras tarefas, como web scraping . Muitas vezes, é melhor transferir essas solicitações para um processo diferente.

* **Análise de dados**: A análise de dados é notoriamente intensiva em recursos. Se seu aplicativo da web analisar dados para seus usuários, você verá rapidamente que seu aplicativo não responde se estiver lidando com todo o trabalho diretamente no Django.

* **Execuções do modelo de aprendizado de máquina**: assim como em outras análises de dados, aguardar os resultados das operações de aprendizado de máquina pode demorar um pouco. Em vez de permitir que seus usuários esperem que os cálculos sejam concluídos, você pode descarregar esse trabalho no Celery para que eles possam continuar navegando em seu aplicativo da web até que os resultados voltem.

* **Geração de relatórios**: se você estiver atendendo a um aplicativo que permite aos usuários gerar relatórios a partir dos dados fornecidos, perceberá que a criação de arquivos PDF não acontece instantaneamente. Será uma experiência de usuário melhor se você deixar o Celery lidar com isso em segundo plano, em vez de congelar seu aplicativo da Web até que o relatório esteja pronto para download.

A configuração principal para todos esses diferentes casos de uso será semelhante. Assim que você entender como transferir processos de computação ou demorados para uma fila de tarefas distribuídas, você liberará o Django para lidar com o ciclo de solicitação-resposta HTTP .

Neste tutorial, você abordará o cenário de envio de e-mail. Você começará com um projeto no qual o Django lida com o envio de e-mail de forma síncrona. Você testará para ver como isso congela seu aplicativo Django. Em seguida, você aprenderá como descarregar a tarefa no Celery para poder experimentar como isso fará com que seu aplicativo da web responda muito mais rapidamente.





# Integre o Celery com o Django
Agora que você sabe o que é o Celery e como ele pode ajudá-lo a melhorar o desempenho do seu aplicativo da web, é hora de integrá-lo para executar tarefas assíncronas com o Celery.

Você se concentrará na integração do Celery em um projeto Django existente. Você começará com um aplicativo Django simplificado com um caso de uso mínimo: coletar feedback do usuário e enviar um e-mail como resposta.

    
### Estrutura de pastas

````shell
source_code_initial/
│
├── django_celery/
│   ├── __init__.py
│   ├── asgi.py
│   ├── settings.py
│   ├── urls.py
│   └── wsgi.py
│
├── feedback/
│   │
│   ├── migrations/
│   │   └── __init__.py
│   │
│   ├── templates/
│   │   │
│   │   └── feedback/
│   │       ├── base.html
│   │       ├── feedback.html
│   │       └── success.html
│   │
│   │
│   ├── __init__.py
│   ├── admin.py
│   ├── apps.py
│   ├── forms.py
│   ├── models.py
│   ├── tests.py
│   ├── urls.py
│   └── views.py
│
├── .gitignore
├── manage.py
└── requirements.txt

````


#### Confirme que você está dentro de source_code_initial/, então crie e ative um ambiente virtual :
Execute os comandos a seguir 
```shell
$ cd source_code_initial

$ python -m venv venv

$ venv\Scripts\activate

(venv) $ pip install django
```

Conclua a configuração local do aplicativo Django executando as migrações e iniciando o servidor de desenvolvimento:
```shell
(venv) $ python manage.py migrate

(venv) $ python manage.py runserver

```
Agora você pode abrir seu navegador para navegar até a página inicial do aplicativo em https://localhost:8000, onde um formulário de feedback de aparência amigável o receberá:

![Tela Inicial](img/tela.png)


Depois de pressionar o botão Enviar , o aplicativo congela. Você pode ver o pequeno símbolo giratório girando na guia do navegador, mas a página não responde e você ainda pode ver todas as informações inseridas no formulário.

Leva muito tempo para o Django processar o formulário e redirecioná-lo para a página de sucesso!

O Django congela porque precisa processar de forma síncrona a solicitação de envio de e-mail antes de realizar a próxima tarefa, que é redirecionar um usuário para a página de sucesso.

A razão pela qual ele congela por tanto tempo é por causa de uma time.sleep()chamada sorrateira .send_email()que simula uma tarefa intensiva de tempo ou trabalho que pode estar associada ao envio de e-mail.

É claro que, em um aplicativo real, você não adicionaria ainda mais atraso ao seu código fazendo o Django dormir. No entanto, qualquer serviço de e-mail que você usar, infelizmente, apresentará alguns atrasos para você. Especialmente quando seu aplicativo começar a atender a muitos usuários, você rapidamente encontrará limitações.

> **Observação**: substitua a time.sleep()chamada por qualquer processo de trabalho intensivo que você precise executar em seu aplicativo da web para atender seus usuários.

Seu aplicativo Django não deve lidar com tarefas de execução longa de forma síncrona, porque isso prejudica a experiência do usuário e a utilidade geral do aplicativo.

Em vez disso, você aprenderá como delegar essa tarefa a um funcionário da Celery. Trabalhadores de aipo podem lidar com cálculos como uma tarefa em segundo plano e permitir que seus usuários continuem navegando em seu aplicativo da web moderno com conteúdo.

####Instale o Celery como sua fila de tarefas

Agora que você configurou o aplicativo de feedback e sentiu o atraso do envio de e-mail, decidiu melhorar a experiência do usuário.

Seu primeiro passo para integrar o Celery ao seu aplicativo Django é instalar o pacote Celery em seu ambiente virtual:

```shell
(venv) $ pip install celery

```
Apenas instalar o Celery, no entanto, não é suficiente. Se você tentar executar a fila de tarefas , notará que primeiro o Celery parece inicializar bem, mas depois exibe uma mensagem de erro que indica que o Celery não consegue encontrar um intermediário de mensagem:

<div style="background-color: #f5f5f5; border: 1px solid #ddd; padding: 10px; margin-bottom: 20px;">

<p>(venv) $ python -m celery worker

[ERROR/MainProcess] consumer: Cannot connect to
⮑ amqp://guest:**@127.0.0.1:5672//: 

[Errno 61] Connection refused.
Trying again in 2.00 seconds... (1/100).</p>
</div>

O Celery precisa de um intermediário de mensagem para se comunicar com programas que enviam tarefas para a fila de tarefas. Sem um corretor, o Celery não consegue receber instruções, e é por isso que continua tentando se reconectar.

<div style="background-color: #f5f5f5; border: 1px solid #ddd; padding: 10px; margin-bottom: 20px;">

<p> <strong> Observação </strong>: você pode observar a sintaxe semelhante à URL no destino ao qual o Celery tenta se conectar. O nome do protocolo, amqp, significa Advanced Message Queuing Protocol e é o protocolo de mensagens que o Celery usa. O projeto mais conhecido que implementa o AMQP nativamente é o RabbitMQ, mas o Redis também pode se comunicar usando o protocolo.</p>
</div>

Antes de usar o Celery, você precisará instalar um agente de mensagens e definir um projeto como produtor de mensagens. No seu caso, o produtor é seu aplicativo Django e o agente de mensagens será o Redis.


Instale o Redis como seu Celery Broker e back-end de banco de dados
Você precisa de um agente de mensagens para que o Celery possa se comunicar com o produtor da tarefa. Você usará o Redis porque o Redis pode servir como um intermediário de mensagens e um back-end de banco de dados ao mesmo tempo.

Volte para o seu terminal e instale o Redis no seu sistema:


````linux
(venv) $ sudo apt update
(venv) $ sudo apt install redis
````

Após a conclusão da instalação, você pode iniciar o servidor Redis para confirmar se tudo funcionou. Abra uma nova janela de terminal para iniciar o servidor:
````linux

$ redis-server
````

Esta janela será sua janela de terminal dedicada para Redis. Mantenha-o aberto para o resto deste tutorial.


<div style="background-color: #f5f5f5; border: 1px solid #ddd; padding: 10px; margin-bottom: 20px;">

<p> <strong> Observação </strong>:a execução redis-server inicia o servidor Redis. Você executa o Redis como um processo independente do Python, portanto, não precisa ter seu ambiente virtual ativado ao iniciá-lo.</p>
</div>



Depois de executar redis-server, seu terminal mostrará o logotipo do Redis como arte ASCII, junto com algumas mensagens de log de inicialização. A mensagem de log mais recente informará que o Redis está pronto para aceitar conexões.

Para testar se a comunicação com o servidor Redis funciona, inicie o Redis CLI em outra nova janela de terminal:


```shell
$ redis-cli
```


Depois que o prompt for alterado, você pode digitar pinge pressionar Entere aguardar a resposta do Redis:

````redis

127.0.0.1:6379> ping
PONG
127.0.0.1:6379>
````


Depois de iniciar a CLI do Redis com redis-cli, você enviou a palavra pingao servidor Redis, que respondeu com um autoritativo PONG. Se você obteve esta resposta, a instalação do Redis foi bem-sucedida e o Celery poderá se comunicar com o Redis.

Saia da CLI do Redis pressionando Ctrl+C antes de passar para a próxima etapa.

Em seguida, você precisará de um cliente Python para interagir com o Redis. Confirme se você está em uma janela de terminal em que seu ambiente virtual ainda está ativo e instale o **redis-py** :

````shell
(venv) $ pip install redis
````

Este comando não instala o Redis em seu sistema, mas apenas fornece uma interface Python para conectar-se ao Redis.

<div style="background-color: #f5f5f5; border: 1px solid #ddd; padding: 10px; margin-bottom: 20px;">

<p> <strong> Observação</strong>:você precisará instalar o Redis em seu sistema e o redis-py em seu ambiente virtual Python para poder trabalhar com o Redis a partir de seus programas Python.</p>
</div>


Depois de concluir ambas as instalações, você configurou com sucesso o message broker. No entanto, você ainda não conectou seu produtor ao Celery.

Se você tentar iniciar o Celery agora e incluir o nome do aplicativo produtor passando a -Aopção junto com o nome do seu aplicativo Django ( django_celery), você se deparará com outro erro:

````shell
(venv) $ python -m celery -A django_celery worker
...
Error: Invalid value for '-A' / '--app':
Unable to load celery application.
Module 'django_celery' has no attribute 'celery'
````


Até agora, sua fila de tarefas distribuídas não pode receber mensagens de seu aplicativo Django porque não há nenhum aplicativo Celery configurado em seu projeto Django.

Na próxima seção, você adicionará o código necessário ao seu aplicativo Django para que ele possa servir como um produtor de tarefas para o Celery.

Adicione Celery ao seu projeto Django
A peça final do quebra-cabeça é conectar o aplicativo Django como um produtor de mensagens à sua fila de tarefas. Você começará com o código do projeto fornecido, então vá em frente e baixe-o se ainda não o fez:

Depois de ter o código do projeto em seu computador, navegue até a django_celerypasta do aplicativo de gerenciamento e crie um novo arquivo chamado celery.py:

````shell
django_celery/
├── __init__.py
├── asgi.py
├── celery.py
├── settings.py
├── urls.py
└── wsgi.py
````

A Celery recomenda usar este módulo para definir a instância do aplicativo Celery. Abra o arquivo em seu editor de texto ou IDE favorito e adicione o código necessário:

````python
# django_celery/celery.py

import os
from celery import Celery

os.environ.setdefault("DJANGO_SETTINGS_MODULE", "django_celery.settings")
app = Celery("django_celery")
app.config_from_object("django.conf:settings", namespace="CELERY")
app.autodiscover_tasks()
````

Você só precisa adicionar essas poucas linhas de código ao arquivo. Continue lendo para saber o que cada um deles realiza:

* Linha 3: você importa o módulo interno os, com o qual você pode estar familiarizado ao trabalhar com arquivos . Você o usará na linha 6 para definir uma variável de ambiente.

* Linha 4: Você importa Celerydo celerypacote. Você o usará na linha 7 para criar sua instância do aplicativo Celery .

* Linha 6: Você usa .setdefault() para os.environgarantir que settings.pyo módulo do seu projeto Django seja acessível por meio da "DJANGO_SETTINGS_MODULE"chave.

* Linha 7: você cria a instância do aplicativo Celery e fornece o nome do módulo principal como um argumento. No contexto do seu aplicativo Django, o módulo principal é o aplicativo Django que contém celery.py, então você passa "django_celery".

* Linha 8: Você define o arquivo de configurações do Django como o arquivo de configuração do Celery e fornece um namespace, "CELERY". Você precisará preceder o valor do namespace, seguido por um sublinhado ( _), para cada variável de configuração relacionada ao Celery. Você pode definir um arquivo de configurações diferente, mas manter a configuração do Celery no arquivo de configurações do Django permite que você fique com um único local central para as configurações.

* Linha 9: Você instrui a instância do aplicativo Celery a localizar automaticamente todas as tarefas em cada aplicativo do projeto Django. Isso funciona desde que você se atenha à estrutura de aplicativos reutilizáveis ​​e defina todas as tarefas do Celery para um aplicativo em um tasks.pymódulo dedicado. Você criará e preencherá esse arquivo para seu django_celeryaplicativo quando refatorar o código de envio de e-mail posteriormente.


````python
# django_celery/settings.py

# ...

# Celery settings
CELERY_BROKER_URL = "redis://localhost:6379"
CELERY_RESULT_BACKEND = "redis://localhost:6379"
````













## Referências
* [Asynchronous Tasks with Django and Celery](https://testdriven.io/blog/django-and-celery/)
* [Django e sistema de filas com Celery — Café com Python #021](https://www.youtube.com/watch?v=QMTyaHAiQX8)
