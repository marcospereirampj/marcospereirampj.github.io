---
layout: post
title:  "Virtualenv: Ambiente virtuais isolados em Python"
date:   2017-07-11 21:15:11 +0000
categories: python
image:  /preview.png
---

Um problema constante para desenvolvedores que estão envolvidos em vários projetos é a configuração do ambiente de desenvolvimento. É bastante comum projetos possuírem dependência da mesma biblioteca, porém com versões diferentes.

Além desse problema, existem ambientes em que não temos permissão de instalar pacotes no sistema. Ou, que queremos proteger nossas configurações do sistema das configurações dos projetos.

O que fazer nesses casos? Como manter ambientes diferentes na mesma máquina? Como isolar um projeto do outro?

Para desenvolvedores Python, esse problema é facilmente resolvido com o uso da **virtualenv**.

### O que é virtualenv?

Como dita na [própria documentação](https://virtualenv.pypa.io/en/latest/index.html), **virtualenv** é uma ferramenta para criação de ambientes Python isolados. Ela oferece uma maneira simples de instalar pacotes Python independentes do sistema global. Além de evitar o conflito com pacotes instalados.

### Como trabalhar com virtualenv?

#### 1º Passo: Instalação

A instalação da **virtualenv** é extremamente prática e simples (no ambiente Linux). Para começo, precisamos instalar o gerenciador de pacotes pip (comando testado no Ubuntu).

{% highlight shell %}
$ sudo apt-get install python-pip python-dev build-essential
{% endhighlight %}

Para versões do Ubuntu anteriores a 10.10, o procedimento muda um pouco e pode ser visto [aqui](http://www.saltycrane.com/blog/2010/02/how-install-pip-ubuntu/).

Feita a instalação do pip, podemos instalar a **virtualenv**.

{% highlight shell %}
$ blog@marcospereira:~/$ sudo pip install --upgrade virtualenv
{% endhighlight %}

#### 2º Passo: Criação e Ativação de Ambientes Virtuais

Com a virtualenv instalada, agora podemos começar a criar os ambientes virtuais para projetos Python. Vamos criar um ambiente:

{% highlight shell %}
$ blog@marcospereira:~/$ virtualenv AmbienteVirtual
{% endhighlight %}

Após a execução do comando acima, será criado um novo diretório contendo a estrutura do Python (sites-packages, bin, include, etc). Podemos carregar o ambiente criado executando o arquivo `activate` que fica dentro do diretório bin da nova estrutura criada.

{% highlight shell %}
$ blog@marcospereira:~/$ source AmbienteVirtual/bin/activate
$ (AmbienteVirtual) blog@marcospereira:~/$
{% endhighlight %}

A partir daí você estará trabalhando no ambiente criado e o prompt do seu shell indicará qual é este ambiente. No nosso caso, é o `AmbienteVirtual`.

#### 3º Passo: Instalação de Pacotes

Estando o ambiente virtual ativado, todas as instalações de pacotes Python afetarão somente esse ambiente. Sendo assim, para instalar um pacote, basta utilizar o pip.

{% highlight shell %}
$ (AmbienteVirtual) blog@marcospereira:~/$ pip install NomeDoPacote
{% endhighlight %}

### Como Organizar?

Bem, agora que você já sabe configurar um ambiente virtual, existem algumas práticas que costumo adotar:

Instale somente os pacotes essenciais ao projeto. Muitas vezes instalamos pacotes que deixamos de utilizar. Isso pode dar uma grande dor de cabeça na hora de verificar as dependências do seu projeto.

Então, sempre monitore os pacotes instalados no seu ambiente. Para isso você pode utilizar o comando:

{% highlight shell %}
$ (AmbienteVirtual) blog@marcospereira:~/$ pip freeze
{% endhighlight %}

Para direcionar a saída para um arquivo, utilize:

{% highlight shell %}
$ (AmbienteVirtual) blog@marcospereira:~/$ pip freeze > requirements.txt
{% endhighlight %}

Esse comando gera uma lista de pacotes instalados e suas versões. Em geral, adicionamos ao projeto o arquivo `requirements.txt` com essa listagem para que outros desenvolvedores saibam quais pacotes o projeto necessita.

Para instalar os pacotes a partir de um arquivo `requirements.txt`, utilize o comando:

{% highlight shell %}
$ (AmbienteVirtual) blog@marcospereira:~/$ pip install -r requirements.txt
{% endhighlight %}

Com vários ambientes virtuais instalados, você pode facilitar sua vida através da extensão `virtualenvwrapper`. Alguns das funcionalidades do `virtualenvwrapper` são:

* Organiza todos os ambientes virtuais em um único lugar.
* Wrappers para gerenciar ambientes virtuais (criar, apagar, copiar).
* Usar um único comando para alternar entre ambientes.
* Reconhecimento de comandos para trabalhar com ambientes.

Bom galera, acho que é isso! Espero ter ajudado.

Abraços a todos e até a próxima.
