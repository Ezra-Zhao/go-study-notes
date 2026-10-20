# Python 游戏编程入门笔记：pygame 基础

## pygame 是什么

pygame 是 Python 里最常用的 2D 游戏开发库，它在 SDL 多媒体库之上做了一层 Python 封装：开窗口、画图、播声音、读键盘鼠标，都不用碰底层细节。对初学者来说，它的价值在于概念完整——窗口、事件、帧、精灵、碰撞这些游戏开发的核心概念，一个小游戏里就能全部碰到。

安装就一句话：`pip install pygame`。

## 一个游戏的骨架：主循环

所有游戏（不只是 pygame）都长着同一个骨架：

1. **处理事件**：玩家按键、点鼠标、关窗口，都以事件的形式排队等处理。
2. **更新状态**：根据输入和规则，算出这一瞬间每个对象的新位置、新状态。
3. **绘制画面**：把更新后的状态画到屏幕上。
4. **控制帧率**：让上面三步以固定节奏重复（常见 60 次/秒），快了游戏会失控，慢了会卡顿。

这个"事件 → 更新 → 绘制"的循环就是**主循环（game loop）**，是理解一切游戏代码的钥匙。下面这段代码是本文的完整示例：一个可以用左右方向键移动的挡板，接住不断落下的下落物。代码刻意保持短小，但骨架是完整的。

```python
import os
import random
import sys

import pygame

WIDTH, HEIGHT = 640, 480
PADDLE_W, PADDLE_H = 90, 12
BALL_R = 10
FPS = 60


def main():
    pygame.init()
    screen = pygame.display.set_mode((WIDTH, HEIGHT))
    pygame.display.set_caption("接落物小游戏")
    clock = pygame.time.Clock()
    font = pygame.font.Font(None, 36)

    paddle = pygame.Rect(WIDTH // 2 - PADDLE_W // 2, HEIGHT - 30, PADDLE_W, PADDLE_H)
    ball = pygame.Rect(random.randint(20, WIDTH - 20), 0, BALL_R * 2, BALL_R * 2)
    ball_speed = 4
    score = 0
    missed = 0

    # 无头/自动测试时可设置环境变量 PYGAME_DEMO_FRAMES=N，让程序跑满 N 帧后自动退出
    max_frames = int(os.environ.get("PYGAME_DEMO_FRAMES", "0"))
    frames = 0

    running = True
    while running:
        # 1. 处理事件
        for event in pygame.event.get():
            if event.type == pygame.QUIT:
                running = False
        keys = pygame.key.get_pressed()
        if keys[pygame.K_LEFT]:
            paddle.x -= 6
        if keys[pygame.K_RIGHT]:
            paddle.x += 6
        paddle.clamp_ip(screen.get_rect())  # 挡板不移出屏幕

        # 2. 更新状态
        ball.y += ball_speed
        if ball.colliderect(paddle):  # 碰撞检测：接住了
            score += 1
            ball.x = random.randint(20, WIDTH - 20)
            ball.y = 0
            ball_speed = min(ball_speed + 0.3, 12)
        elif ball.y > HEIGHT:  # 漏接
            missed += 1
            ball.x = random.randint(20, WIDTH - 20)
            ball.y = 0

        # 3. 绘制画面
        screen.fill((20, 20, 40))
        pygame.draw.rect(screen, (80, 200, 120), paddle)
        pygame.draw.circle(screen, (240, 200, 60), ball.center, BALL_R)
        text = font.render(f"score {score}  missed {missed}", True, (230, 230, 230))
        screen.blit(text, (10, 10))
        pygame.display.flip()

        # 4. 控制帧率
        clock.tick(FPS)
        frames += 1
        if max_frames and frames >= max_frames:
            running = False

    pygame.quit()
    print(f"demo done: score={score} missed={missed} frames={frames}")
    sys.exit(0)


if __name__ == "__main__":
    main()
```

在装有桌面的环境里直接 `python 文件名.py` 就能玩；无头环境（服务器、CI）可这样验证它能跑通：

```bash
SDL_VIDEODRIVER=dummy SDL_AUDIODRIVER=dummy PYGAME_DEMO_FRAMES=180 python 文件名.py
```

## 逐个拆概念

### 窗口与坐标系

`pygame.display.set_mode((宽, 高))` 创建窗口并返回一个 Surface（画布）。坐标系原点在**左上角**，x 向右增大，y 向下增大——和数学坐标系 y 轴方向相反，写位置代码时先在脑子里翻过来。

### Surface 与 blit

一切能画的东西都是 Surface：窗口是 Surface，加载的图片是 Surface，渲染出的文字也是 Surface。把一张 Surface 画到另一张上叫 **blit**（`目标.blit(源, 位置)`），是每帧绘制的基本动作。`pygame.draw.*` 系列则直接在 Surface 上画线条和几何图形，适合原型阶段。

### Rect：位置与碰撞都靠它

`pygame.Rect(x, y, w, h)` 表示一个矩形区域。pygame 里移动对象、判断碰撞，九成靠 Rect：

- `rect.x / rect.y`、`rect.center` 等属性可直接读写，比自己维护坐标省事；
- `a.colliderect(b)` 判断两个矩形是否重叠，这就是最基础的**碰撞检测**；
- `rect.clamp_ip(边界)` 把矩形限制在边界内，示例里挡板不出界靠的就是它。

### 事件与键盘

事件分两类读法：`pygame.event.get()` 取走事件队列（关窗口、按键按下/松开、鼠标点击都是一次性事件），必须每帧调用，否则队列堆积、窗口还会假死；`pygame.key.get_pressed()` 查的是"此刻哪些键正被按住"的持续状态，适合移动控制。示例里关窗口用前者、方向键移动用后者，各取所长。

### 精灵 Sprite

对象一多（几十个敌人、子弹、道具），逐个维护 Rect 会很乱。pygame 提供 `pygame.sprite.Sprite` 和 `Group`：把每个对象封成 Sprite（自带 image 和 rect），放进 Group 后可统一 `update()`、`draw()`，碰撞也有现成的 `spritecollide`、`groupcollide`。示例只有两个对象所以没用上，但真实项目应该从一开始就按 Sprite 组织。

### 帧率与时间

`clock.tick(60)` 的作用是"这一帧不足 1/60 秒就等一会儿"，让游戏速度与机器性能脱钩。进阶写法是用 `tick()` 返回的毫秒数算 dt（时间增量），按"速度 × dt"移动对象，这样帧率波动时手感不变。初学阶段固定帧率 + 每帧固定步进就够用。

### 音效

`pygame.mixer` 负责声音：`pygame.mixer.Sound("hit.wav")` 加载短音效，碰撞时 `.play()` 播放；背景音乐用 `pygame.mixer.music.load()` + `play(-1)` 循环。注意音效文件要自己准备（wav/ogg 均可），代码里加载不存在的文件会直接报错。

## 初学常见坑

1. **忘了每帧取事件**：窗口不响应、系统判定程序无响应。每帧第一件事就是清空事件队列。
2. **在主循环里加载图片/字体**：每帧重新读盘会卡到没法玩。资源在循环开始前加载一次。
3. **用 float 坐标又直接塞回 Rect**：Rect 属性是整数，累加小数步进（如示例里球速 4.3）赋给 `rect.y` 时会被截断，低速时对象可能不动。对策：自己用 float 变量存真实位置，每帧再同步到 Rect。
4. **图片没 convert**：`pygame.image.load()` 后调 `.convert()` / `.convert_alpha()` 转换成与窗口一致的像素格式，blit 速度差一个数量级。
5. **把逻辑帧率和显示绑死后还想加复杂物理**：先跑通固定步进，之后再上 dt 方案，不要一上来就两个一起学。

## 小结

- 游戏 = 主循环：事件、更新、绘制、控帧，四步循环往复。
- 位置和碰撞先学 Rect，对象多了再升级到 Sprite/Group。
- 资源预加载、每帧清事件队列、速度与帧率的关系，这三件事想通了，小游戏就不会写得别扭。

本文为原创学习笔记：按公开标准知识编写，未参考同名书籍内容。
