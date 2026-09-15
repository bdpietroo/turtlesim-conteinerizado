# Comandos rápidos

Referência rápida para rodar o turtlesim em container.
O passo a passo completo está no [README.md](README.md).

## Subir o container

```bash
docker compose up
```

A janela da tartaruga deve abrir. Encerre com `Ctrl + C`.

Para rodar em segundo plano, sem prender o terminal:

```bash
docker compose up -d
```

## Controlar a tartaruga

Com o container rodando, abra **outro terminal** e execute:

```bash
docker exec -it turtlesim /ros_entrypoint.sh ros2 run turtlesim turtle_teleop_key
```

Forma equivalente, mais explícita:

```bash
docker exec -it turtlesim bash -c "source /opt/ros/humble/setup.bash && ros2 run turtlesim turtle_teleop_key"
```

> O `/ros_entrypoint.sh` (ou o `source`) é obrigatório: o `docker exec` não passa
> pelo `ENTRYPOINT` da imagem, então o comando `ros2` não existe sem ele.

Esse terminal precisa ficar **em foco** para capturar as teclas:

| Tecla | Ação |
|---|---|
| ← ↑ → ↓ | move a tartaruga |
| `G` `B` `V` `C` `D` `E` `R` `T` | gira para orientações absolutas |
| `F` | cancela a rotação |

## Mover sem teclado

Publica velocidade direto no tópico, fazendo a tartaruga andar em círculo:

```bash
docker exec -it turtlesim /ros_entrypoint.sh \
  ros2 topic pub --rate 1 /turtle1/cmd_vel geometry_msgs/msg/Twist \
  '{linear: {x: 2.0}, angular: {z: 1.0}}'
```

## Encerrar

```bash
docker compose down
```

Para também remover containers de serviços antigos:

```bash
docker compose down --remove-orphans
```

## Diagnóstico

```bash
docker compose config          # valida o docker-compose.yml
docker compose ps              # mostra o que está rodando
docker compose logs            # mostra os logs do container
```

```bash
id -u ; id -g                  # seu UID e GID (usados no "user:" do compose)
echo $DISPLAY                  # o display do X (geralmente :0)
ls -l /tmp/.X11-unix/          # o socket do servidor X
xhost                          # quem está autorizado a conectar no X
```

## Explorar o ROS dentro do container

```bash
docker exec -it turtlesim /ros_entrypoint.sh ros2 pkg executables turtlesim
docker exec -it turtlesim /ros_entrypoint.sh ros2 node list
docker exec -it turtlesim /ros_entrypoint.sh ros2 topic list
docker exec -it turtlesim /ros_entrypoint.sh ros2 topic echo /turtle1/cmd_vel
```

## Se a janela não abrir

```bash
xhost +local:docker            # libera conexões locais no servidor X
docker compose up              # tente novamente
xhost -local:docker            # desfaça quando terminar
```
