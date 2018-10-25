---
layout: post
title: 利用Hubot搭建ChatOps核心组件
date: 2018-10-24T11:32:41+00:00
author: mingkai
permalink: /2018/10/chatops-with-hubot/
categories:
  - devops
tags:
  - devops
---

Hubot 是 Github 开源的一款自动化 Bot 任务管理框架，由 CoffeeScript 语言开发（NodeJS），支持自定义 HuBot 脚本。本文主要介绍 Hubot 的使用，另外 ChatOps 架构设计需要考虑与 MatterMost 等消息服务的集成操作。

### 安装 Hubot

安装之前确保已经安装了 node 及 npm , 执行下面的链接进行安装，如果需要集成其他的 chat 服务的 adapter， 请访问下面的链接查找（这里集成默认的 shell adapter）

```
npm install -g yo generator-hubot
yo hubot --adapter shell
```

如果需要集成 mattermost 消息服务，使用 matteruser 适配器

设置环境变量

```javascript
export MATTERMOST_HOST=
export MATTERMOST_GROUP=
export MATTERMOST_USER=
export MATTERMOST_PASSWORD=
```

执行后可以保存修改代码并上传到内部的 git 服务器上.

执行下面的命令进入 shell 交互式操作界面, 需要本地启动 redis 服务器来管理所有的持久化数据，可以直接使用 docker 来启动一个 redis 实例用于测试。

```
./bin/hubot

chatops-bot> chatops-bot help
chatops-bot> chatops-bot adapter - Reply with the adapter
chatops-bot animate me <query> - The same thing as `image me`, except adds a few parameters to try to return an animated GIF instead.
chatops-bot echo <text> - Reply back with <text>
...
```

### Hubot 脚本

Hubot 可以看作是消息执行管理器，如何执行这些发送的消息，可以通过 Hubot 脚本实现，这些脚本用于实现不同的人物，社区已经包含很多开发的脚本可以执行 npm 命令获得脚本列表

```
~/chatops-bot(master) » npm search hubot-scripts mattermost
NAME                      | DESCRIPTION          | AUTHOR          | DATE
mattermost-slashbot       | A Hubot script that… | =wpears         | 2016-07-20
hubot-mattermost-sm       | A hubot script for…  | =panjingping    | 2016-09-04
```

这些脚本可以通过 npm install --save 安装到项目中使用

##### 编写自定义脚本

使用 yo 安装项目后，默认的项目中包含有 scripts 目录，包含了一些开发脚本的例子。

```javascript
  robot.respond /open the (.*) doors/i, (res) ->
    doorType = res.match[1]
    if doorType is "pod bay"
      res.reply "I'm afraid I can't let you do that."
    else
      res.reply "Opening #{doorType} doors"

  robot.hear /I like pie/i, (res) ->
    res.emote "makes a freshly baked pie"
```

上述脚本中我们使用了两个方法来响应用户消息，一种是直接回复，比如当输入 I like pie 后直接返回用户消息。 另一个方法是 respond 这种方法需要声明机器人的名字 比如机器人叫做 maven 则需要用户输入。 另外机器人名称与消息之间不能有任何其他的语句或者词语出现。另外 reply 单独的回复给任何一个输入请求的用户。 而 emto 支持将消息传递到一个房间内（适配器支持的前提下）。

```
maven open the pod bay doors
@maven open the pod bay doors
```

另外我们可以同时定义多个响应消息到指定的目的地，比如下面第一条消息直接传递到指定的房间内，第二个则反馈给发送请求的人。

```javascript
  robot.respond /I don't like Sam-I-am/i, (res) ->
    room =  'joemanager'
    robot.messageRoom room, "Someone does not like Dr. Seus"
    res.reply  "That Sam-I-am\nThat Sam-I-am\nI do not like\nthat Sam-I-am"
```

##### 捕获参数

假如我们需要用户传递一些参数进来，比如获取用户输入 IP 的机器信息，或者查询某一个机器的端口存活等等。
我们可以使用 req.match 来获取参数，执行对应的操作。

```javascript
 robot.respond /open the (.*) doors/i, (res) ->
    doorType = res.match[1]
    if doorType is "pod bay"
      res.reply "I'm afraid I can't let you do that."
    else
      res.reply "Opening #{doorType} doors"
```

##### 发送 HTTP 请求

robot 内置了 http 请求实例，我们可以直接借助于 http 来对于一些服务或者数据进行抓取操作。

```javascript
  data = JSON.stringify({
    foo: 'bar'
  })
  robot.http("https://midnight-train")
    .header('Content-Type', 'application/json')
    .post(data) (err, res, body) ->
      # your code here
```

或者 get:

```javascript
 robot.http("https://midnight-train")
    .get() (err, res, body) ->
      # error checking code here

      res.send "Got back #{body}"
```

如果查询的是 json 数据, 可以使用下面的方法发送请求比如 jwt token 等，获取响应并返回给前端

```javascript
  robot.http("https://midnight-train")
    .header('Accept', 'application/json')
    .get() (err, res, body) ->
      # err & response status checking code here

      if response.getHeader('Content-Type') isnt 'application/json'
        res.send "Didn't get back JSON :("
        return

      data = null
      try
        data = JSON.parse body
      catch error
       res.send "Ran into an error parsing JSON :("
       return

      # your code here
```

##### 随机响应

执行随机响应很容易实现，只需要调用 res.random 即可（传递列表数据）

```javascript
lulz = ['lol', 'rofl', 'lmao']
res.send res.random lulz
```

##### 配置进入和离开的问候语

如果适配器支持的话，hubot 将自动的发送已经配置好的语句，提供人性化的支持。

```javascript
enterReplies = ['Hi', 'Target Acquired', 'Firing', 'Hello friend.', 'Gotcha', 'I see you']
leaveReplies = ['Are you still there?', 'Target lost', 'Searching']

module.exports = (robot) ->
  robot.enter (res) ->
    res.send res.random enterReplies
  robot.leave (res) ->
    res.send res.random leaveReplies
```

##### 自定义匹配

如果需要对于发送的消息进行特殊的定制，比如仅仅回复给特殊的用户等需求，可以使用 listen 来执行操作

```javascript
module.exports = (robot) ->
  robot.listen(
    (message) -> # Match function
      # Occassionally respond to things that Steve says
      message.user.name is "Steve" and Math.random() > 0.8
    (response) -> # Standard listener callback
      # Let Steve know how happy you are that he exists
      response.reply "HI STEVE! YOU'RE MY BEST FRIEND! (but only like #{response.match * 100}% of the time)"
  )
```

##### 环境变量

node 中的环境变量通过 process.env 访问，hubot 中同样使用这种方式来管理环境变量的使用

```javascript
answer = process.env.HUBOT_ANSWER_TO_THE_ULTIMATE_QUESTION_OF_LIFE_THE_UNIVERSE_AND_EVERYTHING

module.exports = (robot) ->
  robot.respond /what is the answer to the ultimate question of life/, (res) ->
    unless answer?
      res.send "Missing HUBOT_ANSWER_TO_THE_ULTIMATE_QUESTION_OF_LIFE_THE_UNIVERSE_AND_EVERYTHING in environment: please set and try again"
      return
    res.send "#{answer}, but what is the question?"
```

##### 设置 timer 操作

可以使用 setInterval 或者 setTimeout 来执行 timer 操作，

```javascript
module.exports = (robot) ->
  annoyIntervalId = null

  robot.respond /annoy me/, (res) ->
    if annoyIntervalId
      res.send "AAAAAAAAAAAEEEEEEEEEEEEEEEEEEEEEEEEIIIIIIIIHHHHHHHHHH"
      return

    res.send "Hey, want to hear the most annoying sound in the world?"
    annoyIntervalId = setInterval () ->
      res.send "AAAAAAAAAAAEEEEEEEEEEEEEEEEEEEEEEEEIIIIIIIIHHHHHHHHHH"
    , 1000

  robot.respond /unannoy me/, (res) ->
    if annoyIntervalId
      res.send "GUYS, GUYS, GUYS!"
      clearInterval(annoyIntervalId)
      annoyIntervalId = null
    else
      res.send "Not annoying you right now, am I?"
```

##### HTTP 监听接口

Hubot 包含了 express 框架的一些 http 操作，因此可以直接在脚本中加入对于接口的操作，比如下面的创建一个 api 接口并且执行响应的操作。默认情况下将在 8080 端口上接收用户 u 的请求

```javascript
module.exports = (robot) ->
  # the expected value of :room is going to vary by adapter, it might be a numeric id, name, token, or some other value
  robot.router.post '/hubot/chatsecrets/:room', (req, res) ->
    room   = req.params.room
    data   = if req.body.payload? then JSON.parse req.body.payload else req.body
    secret = data.secret

    robot.messageRoom room, "I have a secret: #{secret}"

    res.send 'OK'
```

同时将消息传递到消息服务器中展示出来。

##### 事件处理

hubot 支持脚本之间通过事件的形式进行消息的传递， 比如下面的当用户调用 url 访问的时候，将产生一个事件，调用响应的脚本和回复给指定的发送人消息。

```javascript
# src/scripts/github-commits.coffee
module.exports = (robot) ->
  robot.router.post "/hubot/gh-commits", (req, res) ->
    robot.emit "commit", {
        user    : {}, #hubot user object
        repo    : 'https://github.com/github/hubot',
        hash  : '2e1951c089bd865839328592ff673d2f08153643'
    }
# src/scripts/heroku.coffee
module.exports = (robot) ->
  robot.on "commit", (commit) ->
    robot.send commit.user, "Will now deploy #{commit.hash} from #{commit.repo}!"
    #deploy code goes here
```

##### 错误处理

hubot 支持设置错误处理回调函数来接受所有未处理的错误消息。比如下面的脚本：

```javascript
# src/scripts/does-not-compute.coffee
module.exports = (robot) ->
  robot.error (err, res) ->
    robot.logger.error "DOES NOT COMPUTE"

    if res?
      res.reply "DOES NOT COMPUTE"
```

另外我们还可以手动的传递 error 事件到处理器：

```javascript
 robot.hear /midnight train/i, (res) ->
    robot.http("https://midnight-train")
      .get() (err, res, body) ->
        if err
          res.reply "Had problems taking the midnight train"
          robot.emit 'error', err, res
          return
        # rest of code here
```

##### 数据持久化

hubot 内部实现了一个内存存储的 kv 数据存储，调用方法如下面的代码，直接使用 robot.brain 获取接口调用 get 或者 set 来设置

```javascript
robot.respond /have a soda/i, (res) ->
  # Get number of sodas had (coerced to a number).
  sodasHad = robot.brain.get('totalSodas') * 1 or 0

  if sodasHad > 4
    res.reply "I'm too fizzy.."

  else
    res.reply 'Sure!'

    robot.brain.set 'totalSodas', sodasHad+1
robot.respond /sleep it off/i, (res) ->
  robot.brain.set 'totalSodas', 0
  msg.reply 'zzzzz'
```

### 消息中间件

中间件包含三种类型

- 接收中间件
- 监听中间件
- 响应中间件

其中接受中间件将在监听中间件检查消息之前运行一次， 监听中间件运行在每个监听器上对于消息进行处理，响应中间件将运行在每一个消息的响应信息处理流程中。中间件的执行类似于 Express 应用中的 next 和 done 来级联不同的中间件或者中断处理。所有的中间件定义都基于`（context,next,done）`的函数签名结构，其中三个参数中第一个参数包含了传递的消息的上下文，比如可以通过上下文获取到用户传递的消息和用户名。

如果想要终止流程可以调用 done 结束调用，或者调用 next 来执行下面的操作

##### 监听中间件

以下为一个监听中间件，用于记录所有用户的访问

```javascript
robot.listenerMiddleware (context, next, done) ->
    # Log commands
    robot.logger.info "#{context.response.message.user.name} asked me to #{context.response.message.text}"
    # Continue executing middleware
    next()
```

```
chatops-bot> i like pie
chatops-bot> [Mon Mar 26 2018 13:45:47 GMT+0800 (CST)] INFO Shell asked me to i like pie
* makes a freshly baked pie
```

下面是一个限速的中间件，用来对于用户输入的指令信息进行限速操作。比如设定最小的调用间隔，一旦小于间隔则执结束用户的操作：

```javascript
module.exports = (robot) ->
  # Map of listener ID to last time it was executed
  lastExecutedTime = {}

  robot.listenerMiddleware (context, next, done) ->
    try
      # Default to 1s unless listener provides a different minimum period
      minPeriodMs = context.listener.options?.rateLimits?.minPeriodMs? or 1000

      # See if command has been executed recently
      if lastExecutedTime.hasOwnProperty(context.listener.options.id) and
         lastExecutedTime[context.listener.options.id] > Date.now() - minPeriodMs
        # Command is being executed too quickly!
        done()
      else
        next ->
          lastExecutedTime[context.listener.options.id] = Date.now()
          done()
    catch err
      robot.emit('error', err, context.response)
```

使用该中间件的方式则是通过,如下的方式，其中`context.listener.options.id`代表了执行的入口参数

```javascript
module.exports = (robot) ->
  robot.hear /hello/, id: 'my-hello', rateLimits: {minPeriodMs: 10000}, (msg) ->
    # This will execute no faster than once every ten seconds
    msg.reply 'Why, hello there!'
```

##### 接收中间件

下面的中间件用来限制用户的访问，比如设定禁止某一些用户执行某一些操作。

```javascript
BLACKLISTED_USERS = [
  '12345' # Restrict access for a user ID for a contractor
]

robot.receiveMiddleware (context, next, done) ->
  if context.response.message.user.id in BLACKLISTED_USERS
    # Don't process this message further.
    context.response.message.finish()

    # If the message starts with 'hubot' or the alias pattern, this user was
    # explicitly trying to run a command, so respond with an error message.
    if context.response.message.text?.match(robot.respondPattern(''))
      context.response.reply "I'm sorry @#{context.response.message.user.name}, but I'm configured to ignore your commands."

    # Don't process further middleware.
    done()
  else
    next(done)
```
