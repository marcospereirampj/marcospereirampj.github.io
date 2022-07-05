---
layout: post
title:  "Servindo Aplicações Django com NGINX + Supervisor + Gunicorn + Virtualenv"
date:   2017-07-13 22:16:11 +0000
categories: python
image:  /images/posts/arquitetura.png
---

Existem diversas maneiras de disponibilizar aplicações web escritas em **Python**. Atualmente tenho utilizado bastante a solução que combina o uso do `NGINX`, `Supervisor`, `Gunicorn` e `virtualenv`.

Com essa nova solução, encontrei vantagens que envolvem principalmente performance, manutenção e monitoramento dos serviços.

Para entender um pouco mais sobre a configuração dessa solução, devemos entender melhor cada uma dessas ferramentas.

### NGINX

O **NGINX** (lê-se "engine x") é um servidor web open source e um server de proxy reverso para protocolos HTTP, SMTP, POP3 e IMAP, com foco em alta concorrência, performance e baixo uso de memória. Por tratar as requisições web como "event-based web server", o **NGINX** consome menos memória que o Apache, que é baseado no conceito de "process-based server".

Na solução que utilizaremos, o **NGINX** servirá somente como servidor de proxy HTTP e reverso.

### Supervisor

O **Supervisor** é uma ferramenta que permite monitorar e controlar uma série de processos em um sistema operacional **UNIX-like**. Ele utiliza a arquitetura cliente/servidor.

Na solução adotada, o **Supervisor** tem a responsabilidade de iniciar e manter a aplicação **Python** sendo executada.

### Gunicorn

O **Gunicorn** (Green Unicorn) é um servidor **WSGI** para **UNIX** escrito em **Python** puro. É uma ferramenta portada do projeto Ruby's Unicorn, que não possui dependências e é de fácil instalação e uso.

Na solução, o **Gunicorn** deverá executar a aplicação **Python**.

### Virtualenv

Como explicado do post ["Virtualenv: Ambiente virtuais isolados em Python"](http://marcospereirajr.com.br/python/2017/07/11/virtualenv-ambiente-virtuais-isolados-em-python.html), o **virtualenv** é uma ferramenta que permite a criação de ambientes virtuais isolados para projetos **Python**. Ela auxilia na gestão de dependência, garantindo que um projeto não interfira no outro.

# Configurando Serviço

Feita as apresentações, podemos iniciar a configuração da solução. A arquitetura que montaremos é melhor apresentada na imagem a seguir.

![Arquitetura da Solução. Fonte: http://pt.slideshare.net/allysonbarros/apresentao-sobre-a-cosinf](/images/posts/arquitetura.png)

Para facilitar a manutenção da aplicação, deveremos organizar os diretórios do projeto da seguinte forma:

{% highlight shell %}
|-- /var/www/
   |-- {projeto}
      |-- config
      |-- logs
      |-- env
      |-- public
{% endhighlight %}

Os diretórios criados tem os seguintes objetivos:

* `config`: diretório contendo os arquivos de configuração do projeto. Neste diretório estão contidos os arquivos do **Supervisor**, **Gunicorn** e de configuração do projeto.
* `public`: diretório contendo todos os arquivos fontes do projeto.
* `env`: diretório contendo a **virtualenv** do projeto. Cada projeto possui sua **virtualenv**.
* `logs`: diretório contendo os arquivos de log do **NGINX** e **Supervisor**.

### Configuração da virtualenv

O primeiro passo que realizaremos é a configuração da **virtualenv**. Sendo assim, deveremos criar uma virtual dentro do diretório *"env"*. Caso ainda não saiba como fazer, leia a postagem ["Virtualenv: Ambiente virtuais isolados em Python"](http://marcospereirajr.com.br/python/2017/07/11/virtualenv-ambiente-virtuais-isolados-em-python.html).

Posteriormente, instale as dependências do seu projeto utilizando o `pip`.

### Configuração do Gunicorn

Para configurar o **Gunicorn** deveremos primeiro instalá-lo na **virtualenv** do projeto. Para isso utilize o pip da **virtualenv**.

{% highlight shell %}
$ pip install gunicorn
{% endhighlight %}

Agora deveremos criar o arquivo de configuração do **Gunicorn** dentro do diretório *"config"*. Iremos criar o arquivo *"gunicorn_config.conf"*. Nesse arquivo deveremos adicionar o seguinte trecho de código:

{% highlight shell %}
command = '/var/www/{projeto}/env/{nome_da_virtualenv}/bin/gunicorn'
pythonpath = '/var/www/{projeto}/env/{nome_da_virtualenv}/'
bind = '{endereco_porta_configurado_gunicorn}'
workers = 2
{% endhighlight %}

Em resumo, o trecho acima indica o comando que deverá ser executado, o diretório do **Python**, o endereço do socket e o número de processos que deverão ser utilizados. Esse arquivo será utilizado pelo **Supervisor**.

### Configuração do Supervisor

Para configurar o **Supervisor** deveremos criar o arquivo *"supervisor.conf"* no diretório *"config"* e adicionar o seguinte trecho:

{% highlight shell %}
[program:{projeto}]
command=/var/www/{projeto}/env/{nome_da_virtualenv}/bin/gunicorn -c /var/www/{projeto}/config/gunicorn_config.py {wsgi_do_projeto_django}:application
directory=/var/www/{projeto}/public/
stderr_logfile=/var/www/{projeto}/logs/long.err.log
stdout_logfile=/var/www/{projeto}/logs/long.out.log
{% endhighlight %}

O trecho acima indica ao **Supervisor** qual commando será executado, o diretório onde ele será executado e a localização dos arquivos de log/out. Percebam que o comando a ser executado é referente ao **Gunicorn**.

Após criar o arquivo de configuração do **Supervisor**, devemos habilitá-lo. Para isso, deveremos adicionar o caminho do arquivo na seção *"include"* em *"/etc/supervisor/supervisor.conf"*. Posteriormente reinicie o serviço *Supervisor*.

{% highlight shell %}
[include]
files = /var/www/{projeto}/config/supervisor.conf
{% endhighlight %}

### Configuração do NGINX

Por fim, deveremos configurar o **NGINX**. Ele funcionará somente como um proxy reverso.

{% highlight shell %}
  server {
        server_name www.dominio.com;

        access_log /var/www/{projeto}/logs/access.log;
        error_log  /var/www/{projeto}/logs/error.log;

        location / {
                proxy_pass_header Server;
                proxy_set_header Host $http_host;
                proxy_set_header X-Real-IP $remote_addr;
                proxy_set_header X-Scheme $scheme;
                proxy_redirect off;
                proxy_connect_timeout 10;
                proxy_read_timeout 10;
                proxy_pass {endereco_porta_configurado_gunicorn};
                add_header P3P 'CP="ALL DSP COR PSAa PSDa OUR NOR ONL UNI COM NAV"';
        }
  }
{% endhighlight %}

O trecho acima deverá ser adicionado a um arquivo dentro do diretório do **NGINX**, *"/etc/nginx/sites-available/"*.

Logo após deveremos habilitar a configuração criando um link simbólico em *"/etc/nginx/sites-enable/"* para o arquivo que acabamos criar (ou simplesmente adicionar o arquivo criado diretamente no diretório
*"/etc/nginx/sites-enable/"* - mas não recomendo).

{% highlight shell %}
$ ln -s /etc/nginx/sites-available/{projeto}.conf {projeto}.conf
{% endhighlight %}

Na configura do **NGINX** existe uma série de configurações de proxy, tentarei comentar sobre essas
configurações em um outro post.

Bom, é isso ai pessoal! Com as ferramentas e configurações apresentadas o serviço deverá está disponível.

Abraços e até a próxima!

## Referências

* [https://virtualenv.pypa.io/en/stable/](https://virtualenv.pypa.io/en/stable/)
* [http://supervisord.org/](http://supervisord.org/)
* [http://gunicorn.org/](http://gunicorn.org/)
* [https://www.nginx.com/](https://www.nginx.com/)
* [https://www.digitalocean.com/community/tutorials/apache-vs-nginx-practical-considerations](https://www.digitalocean.com/community/tutorials/apache-vs-nginx-practical-considerations) (Valeu [@nandooliveira](https://github.com/nandooliveira))
* [https://www.nginx.com/resources/library/complete-nginx-cookbook/?utm_source=twitter&utm_medium=cpc&utm_campaign=core](https://www.nginx.com/resources/library/complete-nginx-cookbook/?utm_source=twitter&utm_medium=cpc&utm_campaign=core) (Valeu [@tiagocedrim](https://github.com/tiagocedrim))
