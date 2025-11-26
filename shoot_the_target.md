
# 🎯 Игра: **«Целься и Стреляй!»**  
*(Target Shooter)*

> **Суть игры**: Игрок управляет пушкой внизу экрана. По экрану медленно двигаются цели (кружки). Игрок кликает мышью — выпускает снаряд. Если снаряд попадает в цель — она исчезает, и игрок получает очко.  
> **Цель**: набрать как можно больше очков за 30 секунд!

Игра учит:
- Работе со списками объектов
- Удалению объектов во время игры
- Таймерам
- Более сложному взаимодействию между классами

---

## 📦 Шаг 1. Подготовка проекта

1. Создайте новый проект в **libGDX Project Generator**:
   - Имя: `TargetShooter`
   - Пакет: `com.mygdx.shooter`
   - Только **Desktop**
2. Откройте в IntelliJ IDEA

---

## 🖼 Шаг 2. Подготовка изображений

Положите в папку `assets`:

- `cannon.png` — пушка (например, 60×40 пикселей, направлена вверх)
- `projectile.png` — снаряд (маленький кружок, 8×8)
- `target.png` — мишень (круг или звезда, 30×30)

> 💡 Фон лучше сделать прозрачным (формат PNG).

---

## 🧱 Шаг 3. Главный класс `TargetShooterGame`

**Файл**: `core/src/com/mygdx/shooter/TargetShooterGame.java`

```java
package com.mygdx.shooter;

import com.badlogic.gdx.Game;
import com.badlogic.gdx.graphics.OrthographicCamera;
import com.badlogic.gdx.graphics.g2d.SpriteBatch;

public class TargetShooterGame extends Game {
    public SpriteBatch batch;
    public OrthographicCamera camera;

    @Override
    public void create() {
        batch = new SpriteBatch();
        camera = new OrthographicCamera();
        camera.setToOrtho(false, 800, 600);

        setScreen(new GameScreen(this));
    }

    @Override
    public void dispose() {
        batch.dispose();
    }
}
```

---

## 🖥 Шаг 4. Экран игры `GameScreen`

**Файл**: `core/src/com/mygdx/shooter/GameScreen.java`

```java
package com.mygdx.shooter;

import com.badlogic.gdx.*;
import com.badlogic.gdx.graphics.GL20;
import com.badlogic.gdx.Screen;
import com.badlogic.gdx.utils.Array;
import com.badlogic.gdx.utils.TimeUtils;

import java.util.Iterator;

public class GameScreen implements Screen {
    private TargetShooterGame game;
    private Cannon cannon;
    private Array<Projectile> projectiles;
    private Array<Target> targets;
    private int score = 0;
    private float timeLeft = 30.0f; // 30 секунд
    private float targetSpawnTimer = 0;

    public GameScreen(TargetShooterGame game) {
        this.game = game;
        cannon = new Cannon(400 - 30, 20); // по центру внизу
        projectiles = new Array<>();
        targets = new Array<>();
    }

    @Override
    public void render(float delta) {
        update(delta);
        draw();
    }

    private void update(float delta) {
        if (timeLeft <= 0) return;

        timeLeft -= delta;
        targetSpawnTimer += delta;

        // Каждые 1.5 секунды появляется новая цель
        if (targetSpawnTimer >= 1.5f) {
            targets.add(new Target(MathUtils.random(0, 770), 600));
            targetSpawnTimer = 0;
        }

        // Управление пушкой (влево/вправо)
        if (Gdx.input.isKeyPressed(Input.Keys.LEFT)) cannon.moveLeft(delta);
        if (Gdx.input.isKeyPressed(Input.Keys.RIGHT)) cannon.moveRight(delta);

        // Выстрел по клику
        if (Gdx.input.justTouched()) {
            projectiles.add(new Projectile(cannon.x + cannon.width / 2 - 4, cannon.y + cannon.height));
        }

        // Обновление снарядов
        for (Iterator<Projectile> it = projectiles.iterator(); it.hasNext(); ) {
            Projectile p = it.next();
            p.update(delta);
            if (p.y > 600) it.remove(); // улетел за экран
        }

        // Обновление целей
        for (Iterator<Target> it = targets.iterator(); it.hasNext(); ) {
            Target t = it.next();
            t.update(delta);
            if (t.y < -30) it.remove(); // ушла вниз — промах
        }

        // Проверка попаданий
        for (Iterator<Projectile> pit = projectiles.iterator(); pit.hasNext(); ) {
            Projectile p = pit.next();
            for (Iterator<Target> tit = targets.iterator(); tit.hasNext(); ) {
                Target t = tit.next();
                if (p.collidesWith(t)) {
                    pit.remove();
                    tit.remove();
                    score++;
                    break;
                }
            }
        }
    }

    private void draw() {
        Gdx.gl.glClearColor(0.2f, 0.4f, 0.8f, 1); // голубой фон
        Gdx.gl.glClear(GL20.GL_COLOR_BUFFER_BIT);

        game.camera.update();
        game.batch.setProjectionMatrix(game.camera.combined);

        game.batch.begin();
        cannon.draw(game.batch);

        for (Projectile p : projectiles) p.draw(game.batch);
        for (Target t : targets) t.draw(game.batch);

        // Вывод счёта и времени (упрощённо — можно позже добавить BitmapFont)
        game.batch.end();
    }

    @Override public void resize(int width, int height) {}
    @Override public void pause() {}
    @Override public void resume() {}
    @Override public void hide() {}
    @Override public void show() {}
    @Override public void dispose() {
        cannon.dispose();
        for (Projectile p : projectiles) p.dispose();
        for (Target t : targets) t.dispose();
    }
}
```

> 🔔 **Примечание**: Для полного вывода счёта и таймера лучше использовать `BitmapFont`, но для простоты мы пока опускаем — фокус на ООП.

---

## 🔫 Шаг 5. Класс пушки `Cannon`

**Файл**: `core/src/com/mygdx/shooter/Cannon.java`

```java
package com.mygdx.shooter;

import com.badlogic.gdx.Gdx;
import com.badlogic.gdx.graphics.Texture;
import com.badlogic.gdx.graphics.g2d.SpriteBatch;

public class Cannon {
    public float x, y;
    public float width, height;
    private Texture texture;

    public Cannon(float x, float y) {
        this.x = x;
        this.y = y;
        texture = new Texture(Gdx.files.internal("cannon.png"));
        width = texture.getWidth();
        height = texture.getHeight();
    }

    public void moveLeft(float delta) {
        x -= 200 * delta;
        if (x < 0) x = 0;
    }

    public void moveRight(float delta) {
        x += 200 * delta;
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

## 💥 Шаг 6. Класс снаряда `Projectile`

**Файл**: `core/src/com/mygdx/shooter/Projectile.java`

```java
package com.mygdx.shooter;

import com.badlogic.gdx.Gdx;
import com.badlogic.gdx.graphics.Texture;
import com.badlogic.gdx.graphics.g2d.SpriteBatch;

public class Projectile {
    public float x, y;
    public float speed = 300;
    private Texture texture;
    public float width, height;

    public Projectile(float x, float y) {
        this.x = x;
        this.y = y;
        texture = new Texture(Gdx.files.internal("projectile.png"));
        width = texture.getWidth();
        height = texture.getHeight();
    }

    public void update(float delta) {
        y += speed * delta;
    }

    public boolean collidesWith(Target target) {
        return x < target.x + target.width &&
               x + width > target.x &&
               y < target.y + target.height &&
               y + height > target.y;
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

## 🎯 Шаг 7. Класс цели `Target`

**Файл**: `core/src/com/mygdx/shooter/Target.java`

```java
package com.mygdx.shooter;

import com.badlogic.gdx.Gdx;
import com.badlogic.gdx.graphics.Texture;
import com.badlogic.gdx.graphics.g2d.SpriteBatch;

public class Target {
    public float x, y;
    public float speed = 50; // медленно падает
    private Texture texture;
    public float width, height;

    public Target(float x, float y) {
        this.x = x;
        this.y = y;
        texture = new Texture(Gdx.files.internal("target.png"));
        width = texture.getWidth();
        height = texture.getHeight();
    }

    public void update(float delta) {
        y -= speed * delta; // движется вниз
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

## ▶️ Шаг 8. Запуск и игра

1. Нажмите **Run** на `TargetShooterGame`
2. Управляйте пушкой стрелками ← →
3. Кликайте мышью, чтобы стрелять
4. Попадайте в цели! Игра длится 30 секунд

---

## 🧠 Что вы освоили

- Работа с **динамическими списками** (`Array`)
- **Создание и удаление объектов** во время игры
- **Взаимодействие между разными типами объектов** (`Projectile` ↔ `Target`)
- Использование **таймеров** и **ограничения по времени**
- ООП: каждый объект — отдельный класс с поведением

---

## 📝 Дополнительные задания

1. Добавь **звук выстрела** и **взрыва при попадании**
2. Сделай **разные типы целей** (медленные/быстрые, дающие разное количество очков)
3. Добавь **экран завершения** с итоговым счётом
4. Реализуй **перезарядку**: нельзя стрелять чаще, чем раз в 0.3 секунды

---

## ✅ Проверь себя

- [ ] Все изображения в `assets`
- [ ] `TargetShooterGame extends Game`
- [ ] Каждый игровой объект — отдельный класс с `draw()` и `update()`
- [ ] Используется общий `SpriteBatch` из главного класса
- [ ] Объекты удаляются из массивов, чтобы не тормозить игру

---

🎮 **Поздравляем!** Теперь у вас есть вторая игра, идеально подходящая для урока по ООП и игровой логике.
