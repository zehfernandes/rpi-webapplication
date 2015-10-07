# Como rodar uma aplicação web na raspberry a 60FPS

_Soon English version_

Raspberry é uma placa single-board incrível, compacta, pontente e de baixo custo. Entrou nas graças de quem gosta de um bom DIY (do it yourself). Com 25 dólares você tem um processador 900MHz quad-core ARM, 1GB de RAM, saída HDMI, mini-sdcard, entrada USB...

Muito usada com aplicações Java, C, processing... permitindo trabalhar em baixo nível em contato direto com os recursos de hardware. Mas para rodar uma aplicação web você fica dependente de um OS e muito distante destes recursos. É preciso customizar uma distro para tirar proveito da placa, de uma olhada no vídeo abaixo :)

[![Raspberry](https://dl.dropboxusercontent.com/u/8015936/D3/raspa.jpg)](https://dl.dropboxusercontent.com/u/8015936/D3/rasp.mp4)


## Hardware Acceleration e GPU.

O uso mais comum é instalar o [Raspbian](https://www.raspbian.org/), distro especial do linux para a Raspberry e abrir sua aplicação no browser disponível (Epiphany ou Midori). Você vai conseguir navegar, mas esqueça o uso de animações e alta performace:
- Primeiro porque a memória de 1GB da RPI é a mesma para vídeo e RAM, ou seja, você possui 1GB divido, se você necessitar de mais RAM para rodar o sistema operacional significa que está perdendo memória de vídeo.
- Segundo e principal motivo, é que nativamente nenhum browser consegue utilizar a GPU (hardware acceleration) para processar as animações.

Mas existe um hack utilizando a biblioteca [QT](https://en.wikipedia.org/wiki/Qt_(software)) para forçar o uso da GPU. Copilando o webkit dentro da biblioteca QT e dando acesso a este recurso.

Para isso é preciso construir uma distro clean do linux, aliviando o máximo da memória RAM e copilando o essencial para poder rodar sua aplicação.


## Buildroot am linux distro

Crosscompile, kernel configuration, library, make são keywords do mundo linux a pouca documentação exata e bem baixo nível comparado ao nosso mundo web. Porém existem umas ferramentas que facilitam esse processo. O buildroot é um jeito simples de conseguir gerar sua distro para ambientes embarcados.

![Image](http://cellux.github.io/articles/diy-linux-with-buildroot-part-1/buildroot.png)

```
Tip: cltr + / : pesquisa
```

Mesmo com ele, é preciso entender qual a dependência de cada sistema e biblioteca que está sendo instalada.
Mas a empresa [Methorogical](https://github.com/Metrological/buildroot) fez um exelente trabalho, iniciado pelo [@albertd](https://github.com/albertd) que montou em cima do buildroot as configs mais cleans possiveis para poder ter a biblioteca QT, webkit e browser instalados.

Basta clonar o repositório:

```sh
git clone https://github.com/Metrological/buildroot
cd buildroot
```

e dentro dele aplicar as configurações básicas para RPI modelo 2

```sh
make rpi2_qt5webkit_defconfig
```

caso queria acessar o menu do builroot e ver possibilidades de instalação, use:

```sh
make menuconfig
```

Neste reposítorio tem um arquivo básico de [.config](https://github.com/zehfernandes/rpi-webapplication/blob/master/snippets/.config) com todas as bibliotecas que achamos ensencial para rodar nossa aplicação. Isso inclui git, fbv, websocket, python...

basta colar no diretório do seu build e rodar

```sh
make
```

O processo leva bastante tempo, principalmente a primeira vez, rodando em um Macbook Air com linux demorou por volta de 3 horas.


## Na raspberry

Terminado a copilação ele vai gerar uma imagem pronta para ser copiada para o SDcard.
[Neste link](https://github.com/Metrological/buildroot#deploying-on-a-raspberry-pi-2) você tem o processo para copiar a imagem via terminal, só seguir os passos.

Com o sdcard pronto, insira ele na RPI, ligue os cabos, o de ethernet também porque com ele, via ssh você vai acessar sua Rasp (login: root, password: root)

```sh
ssh root@192.168.1.100 # replace with RPI ip address
```

E Para rodar sua aplicação dentro da distro:

```sh
qtbrowser --url=http://url
```

## Qtbrowser

O QT não é o chrome e nem o safari, todos usam a engine webkit para renderização, mas possuem detalhes e formas diferentes de lidar com o HTML. É bem importante desenvolver seu front-end testando diretamente no QTbrowser. Durante nossos testes para criação desta interface:

[![Interface](https://dl.dropboxusercontent.com/u/8015936/D3/interface.png)](https://dl.dropboxusercontent.com/u/8015936/D3/rpi-interface.mp4)

Apreendemos vários detalhes e algumas dicas:

- Classes com animações CSS de inicialização não funcionam, é preciso adicionar a classe pós carregar a página.
- Utilize apenas as propriedades que utilizam composite layer
`transition`, `opacity`
- Imagens com dimensões maiores que 1000px tendem a diminuir o FPS
- You don't need jquery
- Evite gradientes e imagens com opacidade
- Várias imagens na tela tem melhor performace que vários canvas
- É melhor clonar seus elementos do DOM que crialos inteirmaente via javascript.


_OBS: Rodamos nossa aplicação em localhost utilizando o [tornado web server](http://www.tornadoweb.org/en/stable/) do python, mas você pode utilizar php ou até mesmo o grunt para levantar seu server._


## Detalhes e sh

A distro criada a partir do buildroot ela é clean a ponto de muitos recursos que um usuário linux está acostumado utilizar não existem. Por exemplo o `apt-get`. As vezes é preciso instalar manualmente alguns recursos.


### Configurações da RPI

Dentro da partição `boot` você tem o arquivo [config.txt](https://github.com/zehfernandes/rpi-webapplication/blob/master/snippets/config.txt) que define valores básicos para o hardware da Raspberry. Alguns valores modificados para este projeto:

```txt
gpu_mem_1024=512 # metade da mémoria para vídeo
hdmi_group=1
hdmi_mode=16 # Full HD 60p
hdmi_force_hotplug=1 # Força entrada HDMI
```


### Inicialização automática

O legal de utilizar a RPI é montar um sistema intermitente, que se atualize e inicie sozinho. Utilizando sh conseguimos criar alguns snippts que ajudam nisto.

Dentro da pasta `/etc/init.d/` você possui arquivos nomeados com S01, S02 o número representa a ordem que ele será executado durante o Boot:

- [S80init](https://github.com/zehfernandes/rpi-webapplication/blob/master/snippets/S80init)
- [S90app](https://github.com/zehfernandes/rpi-webapplication/blob/master/snippets/S90apps)


### Splashscreen? Porque não?

O [@felipesanches](https://github.com/felipesanches) desenvolveu um sistema para você ter uma splashscreen. São 3 passos:

1. Faça uma sequência de PNGs com o nome `frame*.png`, sendo o asterisco o número de cada frame e coloque em alguma pasta da sua raspberry
2. Edite o arquivo `S01logging` e nas primeiras linhas coloque o seguinte código:
```sh
#early splash!
cat /dev/zero 1> /dev/fb0 2>/dev/null
fbv -i -c /home/default/bootanimations/frame*.png --delay 1
```
3. Use o arquivo [cmdline.txt](https://github.com/zehfernandes/rpi-webapplication/blob/master/snippets/cmdline.txt) deste repositório, ele está preparado para silenciar os logs de incialização. Recomendo fazer isto no final do processo. Eles são utéis durante o processo de desenvolvimento (há)


## Fim

Durante esse processo em descobrir como ter uma aplicação web performática na RPI, apreendi bastante e gostaria de agradescer ao [@netoarmando](https://github.com/netoarmando), [@felipesanches](https://github.com/felipesanches) pelas dicas e pelo intensivo de comandos do terminal.

Caso tenha alguma dúvida sinta-se a vontade em usar os [issues do github](https://github.com/zehfernandes/rpi-webapplication/issues) :D