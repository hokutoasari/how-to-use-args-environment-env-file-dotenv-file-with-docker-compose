# docker-compose コマンドでの args:, environment:, env_file: 及び .env ファイルの使い方

docker-compose v3 での、 `args:`, `environment:`, `env_file:` の使い方と `.env` ファイルの関係で迷子になることがあったので自分用に整理した。

## まとめ

`args:` について

- `build 用変数`だよ
- なので、 `run` 時は存在してないよ
- 同じ変数を宣言した場合 `docker-compose.yml > Dockerfile` だよ
- `docker-compose.yml` で宣言すると `Dockerfile` 内で宣言したのと同等になるよ

`environment:` について

- 環境変数を設定できるよ
- `run` 時に `docker-compose.yml` で宣言した変数が追加・上書きされるよ
- `Dockerfile` の `ENV` 命令で宣言したものは、 `build` 時に `ARG` 命令の変数の様に使えてしまうので注意が必要だよ
- かつ、 `build` 時は、上述の様に `docker-compose.yml` で宣言した変数は渡っていないので注意が必要だよ

`env_file:` について

- ファイルから環境変数を追加する仕組みだよ
- 同名の環境変数が存在する場合の優先度は `environment: > env_file:` だよ
- 複数の `env_file:` を指定した場合は、リストの後者の値で追加・上書きされるよ

`.env` について

- `docker-compose` が利用する特殊なファイルが `.env` だよ
- 変数置換機能を便利にしてくれるファイルが `.env` だよ
- (おそらく) `.env` というファイル名を別名にして利用することは出来ないよ
- `.env` で宣言・定義した変数が `docker-compose.yml` 上で変数として利用できる仕組みだよ

## args: の使い方

まずは、公式ドキュメント。

https://docs.docker.com/compose/compose-file/#args

> Add build arguments, which are environment variables accessible only during the build process.

意訳: ビルド用の変数を設定できます。

ビルド用変数を設定できる仕組み。

※ `environment variables` が後述の `environment:` で指定する、所謂 `環境変数` と紛らわしいので、ここでは `ビルド用変数` と読み替えた。

ARG_VALUE1 - ARG_VALUE4 までを以下のように宣言・値設定して、振る舞いを確認します。

|            | Dockerfile         | docker-compose.yml |
|:----------:|:-------------------|:-------------------|
| ARG_VALUE1 | arg1_in_Dockerfile |                    |
| ARG_VALUE2 | arg2_in_Dockerfile | arg2_in_yml        |
| ARG_VALUE3 |                    | arg3_in_yml        |
| ARG_VALUE4 |                    |                    |

*./args/Dockerfile*

```./args/Dockerfile
FROM busybox

ARG ARG_VALUE1="arg1_in_Dockerfile."
ARG ARG_VALUE2="arg2_in_Dockerfile."

RUN echo "ARG_VALUE1 is ${ARG_VALUE1}" \
  && echo "ARG_VALUE2 is ${ARG_VALUE2}" \
  && echo "ARG_VALUE3 is ${ARG_VALUE3}" \
  && echo "ARG_VALUE4 is ${ARG_VALUE4}"
RUN echo "ARG_VALUE1 is $ARG_VALUE1" >> /tmp/outs.txt \
  && echo "ARG_VALUE2 is $ARG_VALUE2" >> /tmp/outs.txt \
  && echo "ARG_VALUE3 is $ARG_VALUE3" >> /tmp/outs.txt \
  && echo "ARG_VALUE4 is $ARG_VALUE4" >> /tmp/outs.txt

CMD cat /tmp/outs.txt \
  && echo "-----" \
  && echo "ARG_VALUE1 is ${ARG_VALUE1}" \
  && echo "ARG_VALUE2 is ${ARG_VALUE2}" \
  && echo "ARG_VALUE3 is ${ARG_VALUE3}" \
  && echo "ARG_VALUE4 is ${ARG_VALUE4}"

```

*./args/docker-compose.yml*

```./args/docker-compose.yml
version: '3'
services:
  with-args:
    build:
      context: ./
      args:
        ARG_VALUE2: "arg2_in_yml"
        ARG_VALUE3: "arg3_in_yml"

```

実行。

```sh
% docker-compose build --no-cache && docker-compose up
...略...
Step 4/6 : RUN echo "ARG_VALUE1 is ${ARG_VALUE1}"   && echo "ARG_VALUE2 is ${ARG_VALUE2}"   && echo "ARG_VALUE3 is ${ARG_VALUE3}"   && echo "ARG_VALUE4 is ${ARG_VALUE4}"
 ---> Running in 64893f52d5bc
ARG_VALUE1 is arg1_in_Dockerfile.
ARG_VALUE2 is arg2_in_yml
ARG_VALUE3 is
ARG_VALUE4 is
Removing intermediate container 64893f52d5bc
 ---> a66e7626d5eb
...略...
[Warning] One or more build-args [ARG_VALUE3] were not consumed
...略...
with-args_1  | ARG_VALUE1 is arg1_in_Dockerfile.
with-args_1  | ARG_VALUE2 is arg2_in_yml
with-args_1  | ARG_VALUE3 is
with-args_1  | ARG_VALUE4 is
with-args_1  | -----
with-args_1  | ARG_VALUE1 is
with-args_1  | ARG_VALUE2 is
with-args_1  | ARG_VALUE3 is
with-args_1  | ARG_VALUE4 is
args_with-args_1 exited with code 0
```

結果。

|            | Dockerfile         | docker-compose.yml | build 時           | run 時              |
|:----------:|:-------------------|:-------------------|:-------------------|:-------------------|
| ARG_VALUE1 | arg1_in_Dockerfile |                    | arg1_in_Dockerfile |                    |
| ARG_VALUE2 | arg2_in_Dockerfile | arg2_in_yml        | arg2_in_yml        |                    |
| ARG_VALUE3 |                    | arg3_in_yml        | \[Warning\]        |                    |
| ARG_VALUE4 |                    |                    |                    |                    |

- `docker-compose.yml` の `args:` で値は上書きされる
- `docker-compose.yml` で宣言した `args:` で未使用の変数があると Warning となる
- `Dockerfile` で利用しているが、 `docker-compose.yml` で宣言していない変数は警告などは無い
- 当然ながら、 `run` 時には利用できない

## environment: の使い方

まずは、公式ドキュメント。

https://docs.docker.com/compose/compose-file/#environment

> Add environment variables.

環境変数を追加できる仕組み。

前述の `ビルド用変数` とは違い、こちらは所謂 `環境変数` を設定する為の命令。

ENV_VALUE1 - ENV_VALUE4 までを以下のように宣言・値設定して、振る舞いを確認。

|            | Dockerfile         | docker-compose.yml |
|:----------:|:-------------------|:-------------------|
| ENV_VALUE1 | env1_in_Dockerfile |                    |
| ENV_VALUE2 | env2_in_Dockerfile | env2_in_yml        |
| ENV_VALUE3 |                    | env3_in_yml        |
| ENV_VALUE4 |                    |                    |

*./environment/Dockerfile*

```./environment/Dockerfile*
FROM busybox

ENV ENV_VALUE1="env1_in_Dockerfile."
ENV ENV_VALUE2="env2_in_Dockerfile."

RUN echo "ENV_VALUE1 is ${ENV_VALUE1}" \
  && echo "ENV_VALUE2 is ${ENV_VALUE2}" \
  && echo "ENV_VALUE3 is ${ENV_VALUE3}" \
  && echo "ENV_VALUE4 is ${ENV_VALUE4}" \
RUN echo "ENV_VALUE1 is $ENV_VALUE1" >> /tmp/outs.txt \
  && echo "ENV_VALUE2 is $ENV_VALUE2" >> /tmp/outs.txt \
  && echo "ENV_VALUE3 is $ENV_VALUE3" >> /tmp/outs.txt \
  && echo "ENV_VALUE4 is $ENV_VALUE4" >> /tmp/outs.txt

CMD cat /tmp/outs.txt \
  && echo "-----" \
  && echo "ENV_VALUE1 is ${ENV_VALUE1}" \
  && echo "ENV_VALUE2 is ${ENV_VALUE2}" \
  && echo "ENV_VALUE3 is ${ENV_VALUE3}" \
  && echo "ENV_VALUE4 is ${ENV_VALUE4}"

```

*./environment/docker-compose.yml*

```./environment/docker-compose.yml
version: '3'
services:
  with-environment:
    environment:
      ENV_VALUE2: "env2_in_yml"
      ENV_VALUE3: "env3_in_yml"

```

実行。

```sh
% docker-compose build --no-cache && docker-compose up
...略...
Step 4/6 : RUN echo "ENV_VALUE1 is ${ENV_VALUE1}"   && echo "ENV_VALUE2 is ${ENV_VALUE2}"   && echo "ENV_VALUE3 is ${ENV_VALUE3}"   && echo "ENV_VALUE4 is ${ENV_VALUE4}"
 ---> Running in bb4ae383c1e7
ENV_VALUE1 is env1_in_Dockerfile.
ENV_VALUE2 is env2_in_Dockerfile.
ENV_VALUE3 is
ENV_VALUE4 is
Removing intermediate container bb4ae383c1e7
 ---> a01b51cd008a
...略...
with-environment_1  | ENV_VALUE1 is env1_in_Dockerfile.
with-environment_1  | ENV_VALUE2 is env2_in_Dockerfile.
with-environment_1  | ENV_VALUE3 is
with-environment_1  | ENV_VALUE4 is
with-environment_1  | -----
with-environment_1  | ENV_VALUE1 is env1_in_Dockerfile.
with-environment_1  | ENV_VALUE2 is env2_in_yml
with-environment_1  | ENV_VALUE3 is env3_in_yml
with-environment_1  | ENV_VALUE4 is
environment_with-environment_1 exited with code 0
```

結果。

|            | Dockerfile         | docker-compose.yml | build 時           | run 時              |
|:----------:|:-------------------|:-------------------|:-------------------|:-------------------|
| ENV_VALUE1 | env1_in_Dockerfile |                    | env1_in_Dockerfile | env1_in_Dockerfile |
| ENV_VALUE2 | env2_in_Dockerfile | env2_in_yml        | env2_in_Dockerfile | env2_in_yml        |
| ENV_VALUE3 |                    | env3_in_yml        |                    | env3_in_yml        |
| ENV_VALUE4 |                    |                    |                    |                    |

- `build` 時は、 `Dockerfile` 記載の値が使われる
- `run` 時に、 `docker-compose.yml` に設定した値が渡され上書きされる
- `ENV` 命令及び `environment:` あくまで、環境変数をセットするものであることを意識して使うこと
- `build` 時に `ENV` 命令で用意した値を `ビルド用変数` 的に使えてしまうので、注意すること

## env_file: の使い方

まずは、公式ドキュメント。

https://docs.docker.com/compose/compose-file/#env_file

> Add environment variables from a file.

所謂 `環境変数` を"ファイル"から追加する仕組み。

ENV_VALUE1 - ENV_VALUE5 までを以下のように宣言・値設定して、振る舞いを確認。

|            | Dockerfile         | docker-compose.yml | some_env.env  |
|:----------:|:-------------------|:-------------------|:--------------|
| ENV_VALUE1 | env1_in_Dockerfile |                    |               |
| ENV_VALUE2 | env2_in_Dockerfile | env2_in_yml        |               |
| ENV_VALUE3 |                    | env3_in_yml        | env3_in_file  |
| ENV_VALUE4 |                    |                    | env4_in_file  |
| ENV_VALUE5 |                    |                    |               |

*./env_file/Dockerfile*

```./env_file/Dockerfile
FROM busybox

ENV ENV_VALUE1="env1_in_Dockerfile."
ENV ENV_VALUE2="env2_in_Dockerfile."

RUN echo "ENV_VALUE1 is ${ENV_VALUE1}" \
  && echo "ENV_VALUE2 is ${ENV_VALUE2}" \
  && echo "ENV_VALUE3 is ${ENV_VALUE3}" \
  && echo "ENV_VALUE4 is ${ENV_VALUE4}" \
  && echo "ENV_VALUE5 is ${ENV_VALUE5}"
RUN echo "ENV_VALUE1 is $ENV_VALUE1" >> /tmp/outs.txt \
  && echo "ENV_VALUE2 is $ENV_VALUE2" >> /tmp/outs.txt \
  && echo "ENV_VALUE3 is $ENV_VALUE3" >> /tmp/outs.txt \
  && echo "ENV_VALUE4 is $ENV_VALUE4" >> /tmp/outs.txt \
  && echo "ENV_VALUE5 is $ENV_VALUE5" >> /tmp/outs.txt

CMD cat /tmp/outs.txt \
  && echo "-----" \
  && echo "ENV_VALUE1 is ${ENV_VALUE1}" \
  && echo "ENV_VALUE2 is ${ENV_VALUE2}" \
  && echo "ENV_VALUE3 is ${ENV_VALUE3}" \
  && echo "ENV_VALUE4 is ${ENV_VALUE4}" \
  && echo "ENV_VALUE5 is ${ENV_VALUE5}"

```

*./env_file/docker-compose.yml*

```./env_file/docker-compose.yml
version: '3'
services:
  with-env_file:
    build:
      context: ./
    environment:
      ENV_VALUE2: "env2_in_yml"
      ENV_VALUE3: "env3_in_yml"
    env_file:
      - some_env.env

```

*./env_file/some_env.env*

```./env_file/some_env.env
ENV_VALUE3="env3_in_file"
ENV_VALUE4="env4_in_file"

```


実行。

```sh
% docker-compose build --no-cache && docker-compose up
...略...
Step 4/6 : RUN echo "ENV_VALUE1 is ${ENV_VALUE1}"   && echo "ENV_VALUE2 is ${ENV_VALUE2}"   && echo "ENV_VALUE3 is ${ENV_VALUE3}"   && echo "ENV_VALUE4 is ${ENV_VALUE4}"   && echo "ENV_VALUE5 is ${ENV_VALUE5}"
 ---> Running in 5851a9b3aa91
ENV_VALUE1 is env1_in_Dockerfile.
ENV_VALUE2 is env2_in_Dockerfile.
ENV_VALUE3 is
ENV_VALUE4 is
ENV_VALUE5 is
Removing intermediate container 5851a9b3aa91
 ---> 39f56354d7cd
...略...
with-env_file_1  | ENV_VALUE1 is env1_in_Dockerfile.
with-env_file_1  | ENV_VALUE2 is env2_in_Dockerfile.
with-env_file_1  | ENV_VALUE3 is
with-env_file_1  | ENV_VALUE4 is
with-env_file_1  | ENV_VALUE5 is
with-env_file_1  | -----
with-env_file_1  | ENV_VALUE1 is env1_in_Dockerfile.
with-env_file_1  | ENV_VALUE2 is env2_in_yml
with-env_file_1  | ENV_VALUE3 is env3_in_yml
with-env_file_1  | ENV_VALUE4 is env4_in_file
with-env_file_1  | ENV_VALUE5 is
env_file_with-env_file_1 exited with code 0
```

結果。

|            | Dockerfile         | docker-compose.yml | some_env.env  | build 時           | run 時              |
|:----------:|:-------------------|:-------------------|:--------------|:-------------------|:-------------------|
| ENV_VALUE1 | env1_in_Dockerfile |                    |               | env1_in_Dockerfile | env1_in_Dockerfile |
| ENV_VALUE2 | env2_in_Dockerfile | env2_in_yml        |               | env2_in_Dockerfile | env2_in_yml        |
| ENV_VALUE3 |                    | env3_in_yml        | env3_in_file  |                    | env3_in_yml        |
| ENV_VALUE4 |                    |                    | env4_in_file  |                    | env4_in_file       |
| ENV_VALUE5 |                    |                    |               |                    |                    |

- `env_file:` で指定した値よりも `environment:` の値が優先される
- `env_file:` で指定し、 `docker-compose.yml` で未設定の場合は、 `env_file:` で指定した値が利用される
- なお、実験はしてないが、ドキュメントにも記載がある通り、 `env_file:` に複数ファイルを指定すると、リストの後者の値で追加・上書きされる

## .env ファイルの使い方

`.env` ファイルは、 `docker-compose` の中で特別に扱われるファイル。

変数置換機能にて利用される。

### 変数置換機能

`.env` ファイルは、 `docker-compose` の変数置換機能にて利用される為、まずは変数置換機能を試す。

公式ドキュメント。 https://docs.docker.com/compose/compose-file/#variable-substitution

> Your configuration options can contain environment variables. Compose uses the variable values from the shell environment in which docker-compose is run.

意訳: `docker-compose` を実行するシェルの環境変数を docker-compose の変数として利用する事ができます。

以下の内容で `./variable-substitution/docker-compose.yml` を用意する。

```./variable-substitution/docker-compose.yml
version: '3'
services:
  variable-substitution:
    image: busybox:${BUSYBOX_VERSION}

```

一時的に `BUSYBOX_VERSION=latest` という環境変数を設定しながら `docker-compose up` する。(`build` の必要が無いので、直接 `up` となる。)

```sh
% env BUSYBOX_VERSION="latest" docker-compose up
...略...
variable-substitution_variable-substitution_1 exited with code 0
```

無事に pull して up して終了した。

同様に、 `BUSYBOX_VERSION=musl` を環境変数に設定し、 `docker-compose up` する。

```sh
% env BUSYBOX_VERSION="musl" docker-compose up
...略...
variable-substitution_variable-substitution_1 exited with code 0
```

こちらも同様に無事に pull して up して終了した。

確認のため、 `docker images` で手元の image 一覧を確認する。

```sh
% docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
busybox             musl                8bce8d24824c        5 weeks ago         1.47MB
busybox             latest              f0b02e9d092d        5 weeks ago         1.23MB
```

確かに、 `latest`, `musl` TAG が打たれた image が存在することを確認した。

`docker-compose` には、このように、シェルを実行する環境の環境変数を `docker-compose` に渡す仕組みがある。

これが、変数置換機能の1つ。
 
### .env ファイル

公式ドキュメントには、以下とある。

https://docs.docker.com/compose/compose-file/#variable-substitution

> You can set default values for environment variables using a .env file, which Compose automatically looks for. Values set in the shell environment override those set in the .env file.
  
意訳: `docker-compose` は、 `.env` ファイルを見つけると、デフォルトの環境変数を設定します。また、シェルの環境変数から設定された値は、 `.env` ファイルで設定された値で上書きされます。

以下のように、 `./variable-substitution-dotenv/.env` ファイルを用意する。

```./variable-substitution-dotenv/.env
BUSYBOX_VERSION=latest

```

先程と同じ内容だが、 `./variable-substitution-dotenv/docker-compose.yml` を用意する。

```./variable-substitution-dotenv/docker-compose.yml
version: '3'
services:
  variable-substitution-dotenv:
    image: busybox:${BUSYBOX_VERSION}

```

この状態で、今度は一時的な環境変数を設定せずに `up` してみる。

※事前に作成済みのコンテナを `docker rm` から削除、同様に pull 済みの images も `docker rmi` で削除しておくと動作がわかりやすいかも知れない。

```sh
% docker-compose up
Creating network "variable-substitution-dotenv_default" with the default driver
Pulling variable-substitution-dotenv (busybox:latest)...
latest: Pulling from library/busybox
9758c28807f2: Pull complete
Digest: sha256:a9286defaba7b3a519d585ba0e37d0b2cbee74ebfe590960b0b1d6a5e97d1e1d
Status: Downloaded newer image for busybox:latest
Creating variable-substitution-dotenv_variable-substitution-dotenv_1 ... done
Attaching to variable-substitution-dotenv_variable-substitution-dotenv_1
```

無事に、 `latest` が pull されて、 up したことがわかる。

続けて、 `./variable-substitution-dotenv/.env` を以下の内容に書き換える。

```./variable-substitution-dotev/.env
BUSYBOX_VERSION=musl

```

この状態で、再度 `docker-compose up` してみる。

```sh
% docker-compose up
Pulling variable-substitution-dotenv (busybox:musl)...
musl: Pulling from library/busybox
7c3804618ebb: Pull complete
Digest: sha256:605de95bca536139f324abdecf88dcab492c8457346e7fc92e37dff6e263f341
Status: Downloaded newer image for busybox:musl
Recreating variable-substitution-dotenv_variable-substitution-dotenv_1 ... done
Attaching to variable-substitution-dotenv_variable-substitution-dotenv_1
variable-substitution-dotenv_variable-substitution-dotenv_1 exited with code 0
```

無事に、 `musl` が pull されて、 up したことがわかる。

このように、 `.env` ファイルを用いると、シェル実行環境の環境変数を汚染せずに、 `docker-compose` に変数を渡す事が可能となる。

ちなみに、公式ドキュメントには、以下の注意書きがある。

> Note when using docker stack deploy
> The .env file feature only works when you use the docker-compose up command and does not work with docker stack deploy.

意訳

> ノート: `docker stack deploy` を使う方へ
> `.env` ファイルを用いた本機能は、 `docker-compose up` コマンドのみ動作します。 `docker stack deploy` では、動かないよ。

`docker stack deploy` 利用時の注意書きなので、 `only works when you use the docker-compose up command` と書いてるのだと思うが、実際は `docker-compose up` コマンドだけではなく、 `dockre-compose` コマンドに対して機能する。

具体的には、 `docker-compose build` や `docker-compose config` コマンドでも機能する。

試しに、現在の状態で `docker-compose config` を実行すると下記となる。(`docker-compose config` は、最終的に実行される yml を確認する為のコマンド。)

```sh
% docker-compose config
services:
  variable-substitution:
    image: busybox:musl
version: '3'

```

`${BUSYBOX_VERSION}` 部分が変数展開されて、 `busybox:musl` となっているのがわかる。

続けて、 `./variable-substitution-dotenv/.env` を以下の内容に書き換えて `docker-compose config` を実行すると以下となる。

```./variable-substitution-dotenv/.env
BUSYBOX_VERSION=latest

```

```sh
% docker-compose config
services:
  variable-substitution:
    image: busybox:latest
version: '3'

```

同様に `${BUSYBOX_VERSION}` 部分が変数展開されて、 `busybox:latest` となっているのがわかる。

`.env` ファイルを用いることで `docker-compose` に変数が渡せる事が確認できた。

#### .env ファイルを用いた変数置換機能で勘違いしやすい点と対処法

なお、当然ながら、以下の `.env`, `Dockerfile` 及び `docker-compose.yml` の構成は `build` 出来ない。

*./variable-substitution-dotenv-dockerfile-not-work/.env*

```./variable-substitution-dotenv-dockerfile-not-work/.env
BUSYBOX_VERSION="latest"

```

*./variable-substitution-dotenv-dockerfile-not-work/Dockerfile*

```./variable-substitution-dotenv-dockerfile-not-work/Dockerfile
FROM busybox:${BUSYBOX_VERSION}

```

*./variable-substitution-dotenv-dockerfile-not-work/docker-compose.yml*

```./variable-substitution-dotenv-dockerfile-not-work/docker-compose.yml
version: '3'
services:
  variable-substitution-dotenv-dockerfile-not-work:
    build:
      context: ./

```

実行。

```sh
% docker-compose build --no-cache
Building variable-substitution-dotenv-dockerfile-not-work
Step 1/1 : FROM busybox:${BUSYBOX_VERSION}
ERROR: Service 'variable-substitution-dotenv-dockerfile-not-work' failed to build : invalid reference format
```

これまでの動作確認や公式ドキュメントを読めば当然なのだが、以下の状態となっている。

- `Dockerfile` の冒頭で用いている `busybox:${BUSYBOX_VERSION}` の `BUSYBOX_VERSION` 変数は `build 用変数`として `Dockerfile` として宣言されていない
- `.env` ファイルを用いて `docker-compose` に変数を渡す仕組みは、当然ながら `docker-compose` までしか値が渡らない
- つまり、 `Dockerfile` まで変数が渡っていない

`build` できるようにするには、以下の構成とする必要がある。

*./variable-substitution-dotenv-dockerfile-work/.env*

```./variable-substitution-dotenv-dockerfile-work/.env
BUSYBOX_VERSION="latest"

```

*./variable-substitution-dotenv-dockerfile-work/Dockerfile*

```./variable-substitution-dotenv-dockerfile-work/Dockerfile
ARG BUSYBOX_VERSION
FROM busybox:${BUSYBOX_VERSION}

```

*./variable-substitution-dotenv-dockerfile-work/docker-compose.yml*

```./variable-substitution-dotenv-dockerfile-work/docker-compose.yml
version: '3'
services:
  variable-substitution-dotenv-dockerfile-work:
    build:
      context: ./
      args:
        BUSYBOX_VERSION: ${BUSYBOX_VERSION}

```

実行。

```sh
% docker-compose build --no-cache
Building variable-substitution-dotenv-dockerfile-work
Step 1/2 : ARG BUSYBOX_VERSION
Step 2/2 : FROM busybox:${BUSYBOX_VERSION}
latest: Pulling from library/busybox
9758c28807f2: Pull complete
Digest: sha256:a9286defaba7b3a519d585ba0e37d0b2cbee74ebfe590960b0b1d6a5e97d1e1d
Status: Downloaded newer image for busybox:latest
 ---> f0b02e9d092d

Successfully built f0b02e9d092d
Successfully tagged variable-substitution-dotenv-dockerfile-work_variable-substitution-dotenv-dockerfile-work:latest
```

- `Dockerfile` にて `ARG` 命令を使い `BUSYBOX_VERSION` 変数を宣言
- `docker-compose.yml` の `args:` 節で `BUSYBOX_VERSION` 変数を宣言
- また、 `docker-compose.yml` の `args:` 節で `BUSYBOX_VERSION` 変数宣言に対して、 `.env` の `BUSYBOX_VERSION` 変数を割り当て
- `.env` ファイルにて、 `BUSYBOX_VERSION` 変数の値を設定

これで、 `.env` の中で定義した `BUSYBOX_VERSION` の値を変更することで、シェル実行環境の環境変数を汚染せずに値を変更できるようになる。

ただ、上記の例示だと、 `BUSYBOX_VERSION` という変数名が各所(`.env`, `Dockerfile` 及び `docker-compose.yml`)で用いられていて、少々わかりにくい。

もう少しわかりやすく明示的に書くと、 `.env`, `Dockerfile` 及び `docker-compose.yml` は以下となる。

*./variable-substitution-dotenv-dockerfile-work-easy-to-read/.env*

```./variable-substitution-dotenv-dockerfile-work-easy-to-read/.env
DOCKER_COMPOSER_BUSYBOX_VERSION="latest"

```

*./variable-substitution-dotenv-dockerfile-work-easy-to-read/Dockerfile*

```./variable-substitution-dotenv-dockerfile-work-easy-to-read/Dockerfile
ARG BUSYBOX_VERSION="latest"
FROM busybox:${BUSYBOX_VERSION}

```

*./variable-substitution-dotenv-dockerfile-work-easy-to-read/docker-compose.yml*

```./variable-substitution-dotenv-dockerfile-work-easy-to-read/docker-compose.yml
version: '3'
services:
  variable-substitution-dotenv-dockerfile-work-easy-to-read:
    build:
      context: ./
      args:
        BUSYBOX_VERSION: ${DOCKER_COMPOSER_BUSYBOX_VERSION}

```

- `Dockerfile` にて `ARG` 命令を使い `BUSYBOX_VERSION` 変数を宣言し、かつ、 docker-compose から値が渡されない場合はデフォルト値として `latest` を用いるように宣言
- `docker-compose.yml` の `args:` 節で `BUSYBOX_VERSION` 変数を宣言
- また、 `docker-compose.yml` の `args:` 節で `BUSYBOX_VERSION` 変数宣言に対して、 `.env` の `DOCKER_COMPOSER_BUSYBOX_VERSION` 変数を割り当て
- `.env` ファイルにて、 `DOCKER_COMPOSER_BUSYBOX_VERSION` 変数の値を設定

"変数"や"環境変数"という単語が多くなりややこしいので、具体的な変数名を見ると、何がどこに変数・値として受け継がれていくのかが分かると思う。

名称(ラベル)がややこしいだけで、 `.env` -> `docker-compose.yml` -> `Dockerfile` と、変数名と値を受け渡していく仕組みが分かる。

`.env` ファイルを用いた変数置換機能に関しては、(この表現が正しいか怪しいが...) **`.env` ファイルを用いた変数置換機能は `docker-compose.yml` に変数と値を渡す仕組み** と捉えるとわかりやすいかも知れない。

##### ちょっとした余談

実は、先に例示した以下の書き方はあまり推奨しません。

```Dockerfile
ARG BUSYBOX_VERSION="latest"
FROM busybox:${BUSYBOX_VERSION}

```

例えば、 github actions (等)の制約で、以下のようなケースが、ままあります。

> Dockerfileファイル中の最初の命令はFROMでなければなりません。

https://docs.github.com/ja/free-pro-team@latest/actions/creating-actions/dockerfile-support-for-github-actions#from

そもそも FROM の TAG 値は、(開発|実行)環境の統一性を考えると "latest" すら使わずに、確実な値を使う方が望ましいと考えています。

もちろん、この限りじゃないケースもあると思いますが、 build のタイミングによっては不本意なメジャーバージョンアップが行われてしまうなどの問題があります。

### 活用例

例えば、 `docker-compose build` する際に、ローカル、ステージング、プロダクションなどの環境によって `yarn run {script}` の実行を制御するなどが実現できます。

ローカル、ステージング、プロダクション用のビルド環境が各々別で存在するとし、かつ、それぞれに `.env` ファイルも存在するとします。

*ローカル用 `.env`*

```.env
#...略...
DOCKER_COMPOSE_BUILD_TYPE="dev"
#...略...
```

*ステージング用 `.env`*

```.env
#...略...
DOCKER_COMPOSE_BUILD_TYPE="dev"
#...略...
```

*プロダクション用 `.env`*

```.env
#...略...
DOCKER_COMPOSE_BUILD_TYPE="prod"
#...略...
```

また、これらの `.env` は gitignore されており、リポジトリで管理されていないとします。

この状態で、 `Dockerfile` と `docker-compose.yml` を以下のように用意しておくことで、単一ソースコードを保ちながら、環境ごとの制御が可能となります。

*Dockerfile*

```Dockerfile
#...略...
ARG BUILD_TYPE="local"
#...略...
RUN yarn run ${BUILD_TYPE}
#...略...
```

*docker-compose.yml*

```docker-compose.yml
#...略...
    build:
#...略...
      args:
        BUILD_TYPE: ${DOCKER_COMPOSE_BUILD_TYPE}
#...略...
```

他にも、 `environment:` を使って、各環境ごとのログの向き先やメールドライバの変更、 DB ドライバの変更なども可能です。

php を使った開発なら xdebug 有効無能の設定や、各種設定値を各個人の `.env` 経由で自由に変更可能とするなども実現できます。
