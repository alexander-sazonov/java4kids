
# 🎮 Урок: Создаём игру Pong на Java с libGDX

> **Цель**: Создать игру, в которой шарик отскакивает от стен и ракетки. Если шарик упадёт вниз — игра закончится.  
> **Технологии**: Java, libGDX, изображения (спрайты), объектно-ориентированное программирование.

---

## 📦 Шаг 1. Подготовка проекта

1. Откройте **libGDX Project Generator** (https://libgdx.com/dev/project-setup/)
2. Настройки:
   - **Name**: `PongGame`
   - **Package**: `com.mygdx.pong`
   - Отметьте только **Desktop**
   - Нажмите **Generate**
3. Откройте проект в **IntelliJ IDEA**

> 💡 Убедитесь, что установлена JDK

---

## 🖼 Шаг 2. Подготовка изображений

1. Найдите или создайте два файла:
   - `paddle.png` — прямоугольник 100×10 пикселей (например, зелёный)
   - `ball.png` — квадрат 20×20 пикселей с белым кружком (фон — прозрачный)
2. Положите их в папку **`assets`** 

> 💡 Без этих файлов игра не запустится!

---

## 🧱 Шаг 3. Создаём класс `PongGame`

Этот класс — «хозяин» всей игры. В нём живут главные инструменты: камера и кисть для рисования.

**Файл**: `core/src/com/mygdx/pong/PongGame.java`

```java
package com.mygdx.pong;

import com.badlogic.gdx.Game;
import com.badlogic.gdx.graphics.OrthographicCamera;
import com.badlogic.gdx.graphics.g2d.SpriteBatch;

public class PongGame extends Game {
    public SpriteBatch batch;
    public OrthographicCamera camera;

    @Override
    public void create() {
        batch = new SpriteBatch();
        camera = new OrthographicCamera();
        camera.setToOrtho(false, 800, 600);

        setScreen(new PongGameScreen(this));
    }

    @Override
    public void dispose() {
        batch.dispose();
    }
}
```

---

## 🖥 Шаг 4. Создаём экран игры `PongGameScreen`

Экран — это один «режим» игры (например, уровень, меню). У нас пока один.

**Файл**: `core/src/com/mygdx/pong/PongGameScreen.java`

```java
package com.mygdx.pong;

import com.badlogic.gdx.*;
import com.badlogic.gdx.graphics.GL20;
import com.badlogic.gdx.Screen;

public class PongGameScreen implements Screen {
    private PongGame game;
    private Paddle paddle;
    private Ball ball;
    private boolean gameOver = false;

    public PongGameScreen(PongGame game) {
        this.game = game;
        paddle = new Paddle(800 / 2 - 50, 20);
        ball = new Ball(800 / 2, 600 / 2);
    }

    @Override
    public void render(float delta) {
        if (!gameOver) {
            update(delta);
        }
        draw();

        if (Gdx.input.isKeyJustPressed(Input.Keys.SPACE) && gameOver) {
            dispose();
            game.setScreen(new PongGameScreen(game));
        }
    }

    private void update(float delta) {
        paddle.update();
        ball.update(delta);

        if (ball.isOffScreenBottom()) {
            gameOver = true;
        }

        if (ball.collidesWith(paddle)) {
            ball.bounceOffPaddle(paddle);
        }
    }

    private void draw() {
        Gdx.gl.glClearColor(0, 0, 0, 1);
        Gdx.gl.glClear(GL20.GL_COLOR_BUFFER_BIT);

        game.camera.update();
        game.batch.setProjectionMatrix(game.camera.combined);

        game.batch.begin();
        paddle.draw(game.batch);
        ball.draw(game.batch);
        game.batch.end();
    }

    @Override public void resize(int width, int height) {}
    @Override public void pause() {}
    @Override public void resume() {}
    @Override public void hide() {}
    @Override public void show() {}

    @Override
    public void dispose() {
        paddle.dispose();
        ball.dispose();
    }
}
```

---

## 🎨 Шаг 5. Класс ракетки `Paddle`

Каждый объект теперь сам умеет рисовать себя!

**Файл**: `core/src/com/mygdx/pong/Paddle.java`

```java
package com.mygdx.pong;

import com.badlogic.gdx.Gdx;
import com.badlogic.gdx.graphics.Texture;
import com.badlogic.gdx.graphics.g2d.SpriteBatch;

public class Paddle {
    public float x, y, width, height;
    private Texture texture;

    public Paddle(float startX, float startY) {
        x = startX;
        y = startY;
        texture = new Texture(Gdx.files.internal("paddle.png"));
        width = texture.getWidth();
        height = texture.getHeight();
    }

    public void update() {
        x = Gdx.input.getX() - width / 2;
        if (x < 0) x = 0;
        if (x + width > 800) x = 800 - width;
    }

    public void draw(SpriteBatch batch) {
        batch.draw(texture, x, y);
    }

    public void dispose() {
        texture.dispose();
    }
}
```

---

## ⚽ Шаг 6. Класс шарика `Ball`

**Файл**: `core/src/com/mygdx/pong/Ball.java`

```java
package com.mygdx.pong;

import com.badlogic.gdx.Gdx;
import com.badlogic.gdx.graphics.Texture;
import com.badlogic.gdx.graphics.g2d.SpriteBatch;

public class Ball {
    public float x, y;
    public float speedX = 200, speedY = 200;
    private Texture texture;
    public float diameter;
    public float radius;

    public Ball(float centerX, float centerY) {
        texture = new Texture(Gdx.files.internal("ball.png"));
        diameter = texture.getWidth();
        radius = diameter / 2;
        this.x = centerX - radius;
        this.y = centerY - radius;
    }

    public void update(float delta) {
        x += speedX * delta;
        y += speedY * delta;

        if (x < 0 || x + diameter > 800) speedX = -speedX;
        if (y + diameter > 600) speedY = -speedY;
    }

    public boolean isOffScreenBottom() {
        return y + diameter < 0;
    }

    public boolean collidesWith(Paddle paddle) {
        return y + diameter >= paddle.y &&
               y <= paddle.y + paddle.height &&
               x + diameter >= paddle.x &&
               x <= paddle.x + paddle.width;
    }

    public void bounceOffPaddle(Paddle paddle) {
        speedY = Math.abs(speedY);
        float hitPos = (x + radius - (paddle.x + paddle.width / 2)) / (paddle.width / 2);
        speedX = hitPos * 200;
    }

    public void draw(SpriteBatch batch) {
        batch.draw(texture, x, y);
    }

    public void dispose() {
        texture.dispose();
    }
}
```

---

## ▶️ Шаг 7. Запуск игры

1. Нажмите **Run** на `PongGame` (desktop)
2. Двигайте мышь — ракетка следует за ней
3. Не дайте шарику упасть!
4. После проигрыша нажмите **Пробел**, чтобы начать заново

---

## 🧠 Что мы узнали?

- Как создавать игру по частям с помощью классов
- Что такое `SpriteBatch` и зачем он один на всю игру
- Как объекты могут сами себя рисовать
- Как работает система экранов (`Game` + `Screen`)

---

## 📝 Домашние задания (на выбор)

1. **Добавь счёт**: считай, сколько раз шарик отскочил от ракетки.
2. **Сделай ускорение**: каждый раз после отскока шарик летит быстрее.
3. **Добавь звук**: проигрывай звук при отскоке (`Sound` и `Gdx.audio.newSound`).
4. **Создай экран "Game Over"**: отдельный класс `GameOverScreen`.

---

## ✅ Проверь себя

- [ ] В папке `assets` лежат `paddle.png` и `ball.png`
- [ ] `PongGame` наследуется от `Game`
- [ ] `PongGameScreen` реализует `Screen`
- [ ] `SpriteBatch` и `Camera` созданы **только в `PongGame`**
- [ ] У `Paddle` и `Ball` есть метод `draw(SpriteBatch batch)`

---

**Удачи в разработке игр!** 🚀  
Вы — настоящие разработчики!
