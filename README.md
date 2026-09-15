# Controlar o turtlesim usando container

Enunciado: executar o **turtlesim** (simulador do ROS 2) dentro de um container
Docker e controlar a tartaruga pelo teclado, com a janela aparecendo na tela do
computador.

O desafio principal nao e instalar o ROS: e fazer uma aplicacao grafica que roda
**dentro** do container desenhar na tela do **host**. Container nao tem monitor.

---

## Pre-requisitos

Antes de comecar, confira que a sua maquina atende:

| O que | Como verificar | Esperado |
|---|---|---|
| Linux com ambiente grafico | `echo $XDG_SESSION_TYPE` | `x11` ou `wayland` |
| Servidor X ou XWayland ativo | `echo $DISPLAY` | algo como `:0` |
| Socket do X existe | `ls /tmp/.X11-unix/` | pelo menos um `X0` |
| Docker instalado | `docker --version` | versao 20 ou superior |
| Docker Compose instalado | `docker compose version` | versao 2 ou superior |

Se o `echo $DISPLAY` vier vazio, o tutorial nao vai funcionar: nao ha servidor
grafico para receber a janela.

Se os comandos do Docker nao responderem, siga a **Parte 0** antes de continuar.

Testado em: Ubuntu 22.04 (sessao Wayland com XWayland), Docker 29, ROS 2 Humble.

---

## Caminho mais curto

Se voce so quer o resultado funcionando, sao quatro passos (detalhados na
Parte 2):

1. Descobrir seu UID: `id -u`
2. Criar o `docker-compose.yml` (etapa 2.4)
3. `docker compose up`
4. Em outro terminal: `docker exec -it turtlesim /ros_entrypoint.sh ros2 run turtlesim turtle_teleop_key`

A Parte 1 e opcional e a Parte 2 explica cada decisao do caminho acima.
Se voce ainda nao tem o Docker instalado, comece pela Parte 0.

---

## Parte 0 - Instalar o Docker no Ubuntu

Pule esta parte se o comando abaixo ja responder com uma versao:

```bash
docker --version
```

Fonte oficial: https://docs.docker.com/engine/install/ubuntu/

**Atencao:** nao instale o Docker pelo pacote `docker.io` dos repositorios do
Ubuntu. Ele e uma versao antiga e **nao inclui o `docker compose`**, que este
projeto usa. O procedimento abaixo instala a versao oficial da Docker Inc.

### 0.1 Remover versoes antigas e pacotes conflitantes

```bash
sudo apt remove docker.io docker-compose docker-compose-v2 docker-doc podman-docker containerd runc
```

E normal o `apt` dizer que alguns desses pacotes nao estao instalados. Isso nao
e erro.

### 0.2 Adicionar o repositorio oficial do Docker

Este bloco baixa a chave GPG da Docker e cadastra o repositorio no `apt`. Sem
ele, o `apt` nao conhece os pacotes e o passo 0.3 falha com
"Impossivel encontrar o pacote".

```bash
sudo apt update
sudo apt install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
```

Agora crie o arquivo do repositorio:

```bash
sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF

sudo apt update
```

Os trechos `$( ... )` sao preenchidos automaticamente com a versao do seu
Ubuntu e a arquitetura do seu processador. Nao substitua nada a mao.

### 0.3 Instalar o Docker

```bash
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

O `docker-compose-plugin` e o que fornece o comando `docker compose` (com
espaco). O antigo `docker-compose` (com hifen) nao e usado neste projeto.

### 0.4 Testar

```bash
sudo docker run hello-world
```

Se aparecer a mensagem "Hello from Docker!", a instalacao funcionou.

### 0.5 Usar o Docker sem sudo

Por padrao, so o root usa o Docker. Para rodar os comandos deste tutorial sem
`sudo`:

```bash
sudo groupadd docker          # pode dizer que o grupo ja existe, tudo bem
sudo usermod -aG docker $USER
newgrp docker                 # ou faca logout e login
```

Teste de novo, agora sem `sudo`:

```bash
docker run hello-world
```

> **Nota de seguranca:** estar no grupo `docker` equivale a ter privilegio de
> root na maquina, porque quem controla o Docker pode montar qualquer diretorio
> do sistema dentro de um container. E o procedimento recomendado pela propria
> documentacao para uma estacao de trabalho pessoal, mas vale saber o que
> significa.

### 0.6 Conferir que esta tudo certo

```bash
docker --version
docker compose version
```

Os dois precisam responder. Se o segundo falhar, o `docker-compose-plugin` nao
foi instalado.

---

## Parte 1 - Instalar o ROS 2 nativo (OPCIONAL)

**Voce pode pular esta parte.** O container ja traz o ROS 2 instalado dentro
dele, entao nada aqui e necessario para a Parte 2 funcionar.

Vale fazer por dois motivos:

1. Voce ve o turtlesim funcionando **fora** do Docker. Se depois algo quebrar no
   container, voce sabe que o problema e do container e nao do ROS.
2. Voce entende o que esta dentro da imagem que vai baixar.

Custa cerca de 3 GB de disco e altera o seu sistema (adiciona repositorio apt).
Se a sua maquina e de trabalho e voce nao quer isso, pule para a Parte 2.

### Site para iniciar a instalacao

https://www.ros.org/blog/getting-started/

1. A partir daqui, voce vai selecionar a versao que for compativel com o seu
   sistema operacional. A regra e: cada versao do ROS 2 e feita para uma versao
   especifica do Ubuntu.

   | Ubuntu | Versao do ROS 2 |
   |---|---|
   | 22.04 (Jammy) | Humble |
   | 24.04 (Noble) | Jazzy |

2. Quando a proxima pagina carregar, selecione a opcao **"Ubuntu (deb packges)"**
   no menu lateral.

   Cuidado: existe tambem uma opcao "Ubuntu (source)" dentro de "Alternativas".
   Essa compila tudo do zero e leva horas. Se o sumario da pagina falar em
   "colcon" ou "compile o codigo", voce esta na pagina errada.

3. Siga o passo a passo desde **"Setup Sources"/"Configurar fontes"**.

   Nao pule essa secao. Ela e que cadastra o repositorio do ROS no `apt`. Se
   voce for direto para o `apt install`, o erro sera:

   ```
   E: Impossivel encontrar o pacote ros-humble-desktop
   ```

   Isso significa que o `apt` nao conhece o pacote, porque o repositorio nao foi
   adicionado.

   Observacao sobre o "Set locale": se o seu sistema ja usa UTF-8 (o
   `pt_BR.UTF-8` usa), essa secao e dispensavel. O ROS 2 precisa de UTF-8, nao
   de ingles.

4. Rode os comandos de **"Environment setup"/"Configuracao de ambiente"**.

   Esse `source /opt/ros/humble/setup.bash` precisa ser executado em **cada
   terminal novo**. Ele nao e permanente: ajusta variaveis de ambiente que
   morrem quando o terminal fecha.

5. Depois rode tambem os comandos de **"Try some examples"/"Tente alguns
   exemplos"**.

Para a etapa 5, dois terminais diferentes para cada par de comandos.

O resultado esperado seria algo parecido com isso:

```
Talker
[INFO] [1789477782.622911907] [talker]: Publishing: 'Hello World: 1'
[INFO] [1789477783.623116957] [talker]: Publishing: 'Hello World: 2'
[INFO] [1789477784.623352519] [talker]: Publishing: 'Hello World: 3'
[INFO] [1789477785.623734178] [talker]: Publishing: 'Hello World: 4'
[INFO] [1789477786.623861387] [talker]: Publishing: 'Hello World: 5'

Listener
[INFO] [1789477803.648039105] [listener]: I heard: [Hello World: 22]
[INFO] [1789477804.630113811] [listener]: I heard: [Hello World: 23]
[INFO] [1789477805.630874361] [listener]: I heard: [Hello World: 24]
[INFO] [1789477806.630123704] [listener]: I heard: [Hello World: 25]
[INFO] [1789477807.630702736] [listener]: I heard: [Hello World: 26]
```

O listener comeca a ouvir do numero em que entrou, e nao do 1. Isso e normal:
no ROS 2 quem chega depois nao recebe as mensagens antigas.

Isso confirma que o ROS 2 esta funcionando corretamente.

Depois de obter esses resultados e confirmar que esta tudo certo, pode dar
Ctrl + C nos dois terminais.

6. No menu lateral, procure por **"Tutorials"** e depois **"Beginner: CLI Tools"**.

7. Selecione a opcao **"Using turtlesim, ros2, and rqt"**.

8. Clique na opcao **"Start turtlesim"** e rode o primeiro comando, isso deve
   abrir a tela da tartaruga.

9. Depois, em outro terminal rode o comando da secao **"Use turtlesim"**, com
   isso vai aparecer as teclas que voce deve apertar para controlar a tartaruga.
   Use as setas tambem para fazer a tartaruga andar.

Observacao: a secao "Install rqt" do tutorial pode ser pulada. O rqt e uma
interface grafica de inspecao e nao e usado neste trabalho.

---

## Parte 2 - Rodar o turtlesim em um container

Esta e a parte que cumpre o enunciado.

### 2.1 Levantar os valores da sua maquina

Tres valores do `docker-compose.yml` mudam de computador para computador.
Descubra os seus antes de escrever o arquivo:

```bash
id -u          # seu UID   (geralmente 1000)
id -g          # seu GID   (geralmente 1000)
echo $DISPLAY  # o display (geralmente :0)
```

Anote os tres. Se o seu UID nao for 1000, voce vai precisar ajustar o arquivo.

### 2.2 Escolher a imagem no Docker Hub

1. Abra o Docker Hub: https://hub.docker.com

2. Pesquise por **"ros"**.

3. Clique na opcao da **"Open Source Robotics Foundation"** (o repositorio
   `osrf/ros`, com o selo roxo).

   Atencao: existe tambem uma imagem oficial chamada so `ros`. Ela **nao serve**
   aqui, porque nao possui as tags `-desktop`, que sao as unicas que incluem o
   turtlesim. O link direto e: https://hub.docker.com/r/osrf/ros

4. Va em **Tags** e filtre por uma destas:

   ```
   humble-desktop
   jazzy-desktop
   lyrical-desktop
   ```

   Isso depende de qual voce escolheu no inicio da instalacao, essa escolha foi
   baseada conforme o seu sistema operacional.

   Voce deve selecionar a opcao que realmente contem o "nome-desktop", nada a
   mais e nada menos:

   - `humble-ros-core` e `humble-ros-base` sao enxutas demais e **nao tem**
     turtlesim.
   - `humble-desktop-full` funciona, mas traz simuladores pesados que nao serao
     usados.

5. Agora copie o titulo, no meu caso `osrf/ros:humble-desktop`.

**Observacao importante:** a versao do ROS dentro do container **nao precisa**
corresponder ao Ubuntu do seu computador. A imagem carrega o proprio sistema
operacional dentro dela. Uma imagem `jazzy-desktop` (baseada no Ubuntu 24.04)
roda normalmente num host Ubuntu 22.04. Essa e justamente a vantagem de usar
container.

### 2.3 (Opcional) Baixar a imagem antes

```bash
docker pull osrf/ros:humble-desktop
```

**Este passo e dispensavel.** O `docker compose up` da etapa 2.5 baixa a imagem
automaticamente se ela ainda nao estiver no computador. Rodar o `pull` separado
serve so para ver o download acontecer antes, ja que sao cerca de 1 GB
comprimido (4,8 GB em disco).

Para conferir depois que baixou:

```bash
docker images | grep ros
```

### 2.4 Criar o docker-compose.yml

Crie um arquivo chamado exatamente `docker-compose.yml` na pasta do projeto.

Antes de copiar o conteudo, duas regras de YAML que evitam 90% dos erros:

- A indentacao e feita **so com espacos**. Tabulacao quebra o arquivo.
- Itens de lista comecam com **hifen + espaco** (`- `), um por linha.

Conteudo completo do arquivo:

```yaml
services:
  turtlesim:
    # Imagem escolhida na etapa 2.2.
    # Para ROS 2 Jazzy, troque apenas esta linha por: osrf/ros:jazzy-desktop
    image: osrf/ros:humble-desktop

    # Nome fixo do container, usado depois pelo "docker exec".
    container_name: turtlesim

    # Roda o container com o seu UID/GID (etapa 2.1), para que o servidor X
    # aceite a conexao pela regra SI:localuser que ja existe.
    user: "1000:1000"

    environment:
      # Sem "=valor", o Compose herda o valor do terminal que rodou o comando.
      - "DISPLAY"
      # Desativa a extensao MIT-SHM do X11, que nao funciona entre container
      # e host e faz aplicacoes Qt travarem ou desenharem lixo.
      - "QT_X11_NO_MITSHM=1"
      # O /etc/passwd montado diz que o home do UID 1000 e /home/<usuario>,
      # que nao existe dentro do container. Sem isto o ROS 2 aborta ao tentar
      # criar /home/<usuario>/.ros/log. O /tmp existe e e gravavel.
      - "HOME=/tmp"

    volumes:
      # Socket do servidor X: e por ele que a janela chega na sua tela.
      - /tmp/.X11-unix:/tmp/.X11-unix:rw
      # Permitem resolver o UID 1000 para um usuario com nome dentro do
      # container. Somente leitura.
      - /etc/group:/etc/group:ro
      - /etc/passwd:/etc/passwd:ro

    # Substitui o CMD padrao da imagem (bash). O ENTRYPOINT da imagem
    # (/ros_entrypoint.sh) ja faz o source do setup.bash antes de executar.
    command: ros2 run turtlesim turtlesim_node
```

**Ajustes que talvez voce precise fazer:**

| Se... | Mude |
|---|---|
| seu `id -u` nao for 1000 | a linha `user:` para o seu UID e GID |
| voce usa Jazzy ou outra versao | a linha `image:` |

**Valide antes de rodar:**

```bash
docker compose config
```

Se o comando imprimir o arquivo todo (com os valores ja preenchidos, como
`DISPLAY: :0`), esta correto. Se imprimir um erro, ele diz qual linha ou qual
servico tem problema. Veja a secao "Problemas conhecidos" no fim.

O que cada parte faz esta explicado em detalhe na secao "Como funciona" mais
abaixo.

### 2.5 Subir o container

No terminal, dentro da pasta do projeto:

```bash
docker compose up
```

A janela azul com a tartaruga deve aparecer. No terminal voce vera:

```
turtlesim  | [INFO] [turtlesim]: Starting turtlesim with node name /turtlesim
turtlesim  | [INFO] [turtlesim]: Spawning turtle [turtle1] at x=5.544445, y=5.544445
```

Algumas mensagens aparecem e podem ser ignoradas, desde que a janela abra:

```
QStandardPaths: XDG_RUNTIME_DIR not set, defaulting to '/tmp/runtime-bruno'
MESA: error: Failed to query drm device.
libGL error: failed to load driver: iris
```

As duas ultimas sao porque o container nao tem acesso a placa de video do host e
usa renderizacao por software. Para o turtlesim, que e 2D, nao faz diferenca.

### 2.6 Controlar a tartaruga

Com o container rodando, abra um **segundo terminal** e execute:

```bash
docker exec -it turtlesim /ros_entrypoint.sh ros2 run turtlesim turtle_teleop_key
```

Esse terminal precisa ficar **em foco** para capturar as teclas:

- **setas**: movem a tartaruga
- **G B V C D E R T**: giram para orientacoes absolutas
- **F**: cancela a rotacao

O `/ros_entrypoint.sh` no meio do comando nao e enfeite. Explicacao na secao de
problemas conhecidos.

### 2.7 Encerrar

`Ctrl + C` no primeiro terminal e depois:

```bash
docker compose down
```

Isso para e remove o container e a rede criada.

---

## Como funciona o docker-compose.yml

| Linha | Para que serve |
|---|---|
| `image:` | qual imagem usar. Trocar esta linha muda a versao do ROS |
| `container_name:` | nome fixo do container, para o `docker exec` ficar previsivel |
| `user: "1000:1000"` | roda o container com o seu UID, para o servidor X aceitar a conexao |
| `- "DISPLAY"` | sem `=valor`, o Compose copia o valor do terminal que rodou o comando |
| `- "QT_X11_NO_MITSHM=1"` | desativa uma otimizacao do X11 que nao funciona entre container e host |
| `- "HOME=/tmp"` | aponta o home para um diretorio que existe dentro do container |
| `- /tmp/.X11-unix:...` | monta o socket do servidor X: e por ele que a janela chega na tela |
| `- /etc/passwd`, `/etc/group` | permitem resolver o UID 1000 para um usuario com nome |
| `command:` | o que executar dentro do container |

### Por que o X11 precisa ser configurado

Um servidor grafico X11 escuta num **socket Unix**, que na pratica e um arquivo:
`/tmp/.X11-unix/X0`. O container nao tem tela, mas pode receber um arquivo por
bind mount.

Entao a solucao tem duas partes, e as duas sao necessarias:

1. **Montar o socket** (`volumes:`) - o caminho ate a tela.
2. **Passar a variavel DISPLAY** (`environment:`) - o endereco que diz ao
   programa qual socket procurar.

Com so uma das duas, nao funciona.

### Sobre autorizacao do servidor X

Por padrao o servidor X so aceita conexoes autorizadas. Rodando `xhost` sem
argumentos voce ve a lista:

```
access control enabled, only authorized clients can connect
SI:localuser:bruno
```

A regra `SI:localuser:bruno` aceita qualquer processo que rode com o UID do
usuario `bruno`. Como o container roda com `user: "1000:1000"` (o mesmo UID), a
conexao e aceita **sem precisar afrouxar nada**.

A alternativa, usada na maioria dos tutoriais, e:

```bash
xhost +local:docker
```

Esse comando libera **todas** as conexoes locais, nao so as do Docker. Funciona,
mas e menos seguro. Se voce precisar usar, desfaca depois com:

```bash
xhost -local:docker
```

---

## Problemas conhecidos

### `qt.qpa.xcb: could not connect to display`

Falta a variavel `DISPLAY`, o socket do X11, ou os dois. Confira as secoes
`environment:` e `volumes:` do compose.

Se a mensagem terminar com o display vazio (`could not connect to display `),
a variavel nao chegou. Se aparecer o valor (`display :0`), a variavel chegou e o
problema e o socket ou a autorizacao.

### `Failed to create log directory: /home/<usuario>/.ros/log`

Acontece quando o `/etc/passwd` do host esta montado mas o home nao. O container
descobre que o home do UID 1000 e `/home/<usuario>`, mas esse diretorio nao
existe la dentro.

Solucao: a variavel `HOME=/tmp` no `environment:`, que ja esta no arquivo.

### `ros2: command not found` ao usar `docker exec`

O `docker exec` **nao passa pelo ENTRYPOINT da imagem**. O
`/ros_entrypoint.sh` so roda no processo principal do container. Qualquer
processo extra nasce sem o ambiente do ROS configurado.

Por isso o comando de controle chama o entrypoint explicitamente:

```bash
docker exec -it turtlesim /ros_entrypoint.sh ros2 run turtlesim turtle_teleop_key
```

A forma equivalente, mais explicita:

```bash
docker exec -it turtlesim bash -c "source /opt/ros/humble/setup.bash && ros2 run turtlesim turtle_teleop_key"
```

### `service "x" refers to undefined network` ou `has neither an image nor a build context`

Erros de arquivo compose mal formado, normalmente por copiar um exemplo pela
metade. Valide antes de subir:

```bash
docker compose config
```

Se ele imprimir o arquivo interpretado, esta valido. Se imprimir um erro, ele
diz qual servico e qual o problema.

### Aviso de `orphan containers`

Sobras de servicos que existiam no compose e foram removidos. Limpe com:

```bash
docker compose down --remove-orphans
```

---

## Limitacoes

- **So funciona em Linux com X11 ou XWayland.** O caminho `/tmp/.X11-unix` nao
  existe em macOS nem em Windows. Nesses sistemas seria preciso um servidor X
  externo (XQuartz, VcXsrv) e uma configuracao diferente.
- **O UID esta fixo em 1000.** Em uma maquina onde o usuario nao seja o primeiro
  criado, a linha `user:` precisa ser ajustada a mao. Uma melhoria seria usar
  interpolacao de variavel com um arquivo `.env`.
- **Sem aceleracao grafica.** O container nao acessa a GPU do host, entao o
  OpenGL roda por software. Irrelevante para o turtlesim, relevante para
  simuladores 3D como o Gazebo.
- **Alternativa mais portatil:** o Qt tambem oferece o backend `vnc`, que serve a
  janela pela rede para ser aberta no navegador. Funciona em qualquer sistema
  operacional e ate em servidor sem tela, ao custo de mais complexidade e
  latencia.

---

## Referencias

- Instalacao do Docker no Ubuntu: https://docs.docker.com/engine/install/ubuntu/
- Usar o Docker sem sudo: https://docs.docker.com/engine/install/linux-postinstall/
- Instalacao do ROS 2: https://www.ros.org/blog/getting-started/
- Tutorial oficial do turtlesim: "Tutorials > Beginner: CLI tools > Using
  turtlesim, ros2, and rqt" na documentacao do ROS 2
- Imagens Docker da OSRF: https://hub.docker.com/r/osrf/ros
- Exemplo oficial de ROS com Compose: https://wiki.ros.org/docker/Tutorials/Compose
  (atencao: essa pagina e de ROS 1, com `roscore` e `ROS_MASTER_URI`, que nao
  existem no ROS 2. A secao util e a "Using GUI's with compose")
- Documentacao do Docker Compose: https://docs.docker.com/compose/
