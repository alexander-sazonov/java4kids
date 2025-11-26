
# Проект: «Трасса» (Top-Down Racing)

**Цель:** Создать гонку с видом сверху. Твоя машина находится внизу экрана. Навстречу несутся препятствия (камни) и разметка дороги. Твоя задача — уворачиваться от камней. Чем дольше едешь, тем выше скорость!

**Главный секрет игры:** На самом деле твоя машина **не едет вперед**. Она стоит на месте по вертикали (Y). Иллюзия движения создается тем, что всё остальное (камни и полоски дороги) летит на тебя сверху вниз.

---

## Подготовка

Создай новый проект LibGDX.
Найди или нарисуй 3 картинки и положи их в папку `assets`:
1.  `car.png` — Твоя машина (вид сверху, например 60x100 пикселей).
2.  `stone.png` — Препятствие (камень или другая машина).
3.  `line.png` — Белая полоска разметки дороги (маленький прямоугольник, например 10x40 пикселей).

---

## Шаг 1: Родительский класс (GameObject)

Начнем с классики. Нам нужен общий класс для всех объектов игры.

Создай класс `GameObject`:

```java
import com.badlogic.gdx.graphics.Texture;
import com.badlogic.gdx.graphics.g2d.Batch;
import com.badlogic.gdx.math.Rectangle;

public class GameObject {
    Texture texture;
    float x, y;
    float width, height;
    Rectangle bounds; // Хитбокс для столкновений

    public GameObject(Texture texture, float x, float y) {
        this.texture = texture;
        this.x = x;
        this.y = y;
        this.width = texture.getWidth();
        this.height = texture.getHeight();
        this.bounds = new Rectangle(x, y, width, height);
    }

    public void draw(Batch batch) {
        batch.draw(texture, x, y);
    }

    // Обновляем позицию хитбокса, если объект сдвинулся
    public void updateBounds() {
        bounds.setPosition(x, y);
    }
    
    public Rectangle getBounds() {
        return bounds;
    }
}
```

---

## Шаг 2: Машина игрока (Car)

Машина может двигаться только влево и вправо.

Создай класс `Car`, наследующий `GameObject`:

```java
import com.badlogic.gdx.Gdx;
import com.badlogic.gdx.Input;
import com.badlogic.gdx.graphics.Texture;

public class Car extends GameObject {
    
    float speed = 400; // Скорость перемещения влево-вправо

    public Car(Texture texture) {
        // Ставим машину по центру экрана по горизонтали и немного выше низа экрана
        super(texture, Gdx.graphics.getWidth() / 2 - texture.getWidth() / 2, 20);
    }

    public void update() {
        float dt = Gdx.graphics.getDeltaTime();

        // Управление стрелками
        if (Gdx.input.isKeyPressed(Input.Keys.LEFT)) {
            x -= speed * dt;
        }
        if (Gdx.input.isKeyPressed(Input.Keys.RIGHT)) {
            x += speed * dt;
        }

        // Ограничение: нельзя выезжать за пределы экрана
        // Левая граница
        if (x < 0) x = 0;
        // Правая граница
        if (x > Gdx.graphics.getWidth() - width) x = Gdx.graphics.getWidth() - width;

        updateBounds();
    }
}
```

---

## Шаг 3: Препятствия (Stone)

Камни появляются сверху и летят вниз. Но есть нюанс: скорость падения камня зависит от общей скорости игры (`gameSpeed`), которую мы будем передавать в метод обновления.

Создай класс `Stone`:

```java
import com.badlogic.gdx.Gdx;
import com.badlogic.gdx.graphics.Texture;
import com.badlogic.gdx.math.MathUtils;

public class Stone extends GameObject {

    public Stone(Texture texture) {
        // Появляется за верхней границей экрана
        // Позиция X случайная
        super(texture, 
              MathUtils.random(0, Gdx.graphics.getWidth() - texture.getWidth()), 
              Gdx.graphics.getHeight());
    }

    // В метод update мы передаем gameSpeed — скорость всей гонки
    public void update(float gameSpeed) {
        float dt = Gdx.graphics.getDeltaTime();
        
        // Камень летит вниз со скоростью игры
        y -= gameSpeed * dt;
        
        updateBounds();
    }
    
    // Проверка: улетел ли камень за низ экрана?
    public boolean isOffScreen() {
        return y + height < 0;
    }
}
```

---

## Шаг 4: Разметка дороги (RoadLine)

Чтобы было ощущение скорости, нам нужны мелькающие полоски на асфальте. Они работают точно так же, как камни, но не убивают игрока.

Создай класс `RoadLine`:

```java
import com.badlogic.gdx.Gdx;
import com.badlogic.gdx.graphics.Texture;

public class RoadLine extends GameObject {

    public RoadLine(Texture texture, float x, float y) {
        super(texture, x, y);
    }

    public void update(float gameSpeed) {
        y -= gameSpeed * Gdx.graphics.getDeltaTime();
    }
    
    public boolean isOffScreen() {
        return y + height < 0;
    }
}
```

---

## Шаг 5: Сборка игры (MyGdxGame)

Самое интересное. Здесь мы будем управлять списками объектов и увеличивать скорость игры.

Открой `MyGdxGame` и напиши:

```java
import com.badlogic.gdx.ApplicationAdapter;
import com.badlogic.gdx.Gdx;
import com.badlogic.gdx.graphics.Texture;
import com.badlogic.gdx.graphics.g2d.SpriteBatch;
import com.badlogic.gdx.utils.ScreenUtils;
import java.util.ArrayList;
import java.util.Iterator;

public class MyGdxGame extends ApplicationAdapter {
    SpriteBatch batch;
    
    Texture imgCar, imgStone, imgLine;
    
    Car player;
    
    ArrayList<Stone> stones;
    ArrayList<RoadLine> lines;
    
    // Параметры игры
    float gameSpeed = 300;      // Начальная скорость гонки (пикселей в секунду)
    float stoneSpawnTimer = 0;  // Таймер для камней
    float lineSpawnTimer = 0;   // Таймер для полосок дороги
    int score = 0;              // Очки (пройденная дистанция)

    @Override
    public void create() {
        batch = new SpriteBatch();
        
        imgCar = new Texture("car.png");
        imgStone = new Texture("stone.png");
        imgLine = new Texture("line.png");
        
        player = new Car(imgCar);
        
        stones = new ArrayList<>();
        lines = new ArrayList<>();
    }

    @Override
    public void render() {
        // Очищаем экран серым цветом (цвет асфальта)
        ScreenUtils.clear(0.3f, 0.3f, 0.3f, 1);
        
        float dt = Gdx.graphics.getDeltaTime();

        // --- 1. ЛОГИКА И ОБНОВЛЕНИЯ ---

        // Ускоряем игру со временем!
        // Каждую секунду скорость растет на 5 единиц
        gameSpeed += 5 * dt; 

        player.update();

        // Генерация дорожной разметки (для красоты)
        createRoadLines(dt);
        
        // Генерация препятствий
        createStones(dt);

        // Обновление полосок дороги (удаляем улетевшие)
        Iterator<RoadLine> lineIter = lines.iterator();
        while (lineIter.hasNext()) {
            RoadLine line = lineIter.next();
            line.update(gameSpeed); // Полоски двигаются со скоростью игры
            if (line.isOffScreen()) {
                lineIter.remove();
            }
        }

        // Обновление камней и проверка столкновений
        Iterator<Stone> stoneIter = stones.iterator();
        while (stoneIter.hasNext()) {
            Stone stone = stoneIter.next();
            stone.update(gameSpeed); // Камни тоже летят со скоростью игры

            // Если врезались
            if (stone.getBounds().overlaps(player.getBounds())) {
                System.out.println("БА-БАХ! Игра окончена. Скорость была: " + (int)gameSpeed);
                gameSpeed = 0; // Останавливаем игру (или можно перезапустить: create())
            }

            // Если камень пролетел мимо
            if (stone.isOffScreen()) {
                stoneIter.remove();
                score++; // Добавляем очко за успешный уворот
            }
        }

        // --- 2. ОТРИСОВКА ---
        batch.begin();
        
        // Сначала рисуем дорогу (чтобы она была ПОД машиной)
        for (RoadLine line : lines) {
            line.draw(batch);
        }

        // Рисуем игрока
        player.draw(batch);

        // Рисуем камни (чтобы они были ПОВЕРХ дороги)
        for (Stone stone : stones) {
            stone.draw(batch);
        }
        
        batch.end();
    }
    
    // Вспомогательный метод для создания камней
    private void createStones(float dt) {
        stoneSpawnTimer += dt;
        // Чем быстрее игра, тем чаще появляются камни!
        // Формула: 1.5 сек делим на коэффициент скорости
        if (stoneSpawnTimer > 1.5f * (300 / gameSpeed)) {
            stones.add(new Stone(imgStone));
            stoneSpawnTimer = 0;
        }
    }

    // Вспомогательный метод для создания полосок
    private void createRoadLines(float dt) {
        lineSpawnTimer += dt;
        // Полоски появляются часто, создавая эффект движения
        if (lineSpawnTimer > 0.15f) {
            // Создаем полоску по центру экрана (примерно)
            // Можно создать несколько полос, если дорога широкая
            float screenCenterX = Gdx.graphics.getWidth() / 2;
            lines.add(new RoadLine(imgLine, screenCenterX, Gdx.graphics.getHeight()));
            lineSpawnTimer = 0;
        }
    }

    @Override
    public void dispose() {
        batch.dispose();
        imgCar.dispose();
        imgStone.dispose();
        imgLine.dispose();
    }
}
```

---

## Запуск игры

1.  Запусти проект.
2.  Ты увидишь серый фон, машину и бегущие вниз полоски. Это создает иллюзию, что ты едешь вперед.
3.  Сверху начнут падать камни. Уворачивайся!
4.  Заметь: чем дольше ты играешь, тем быстрее движутся полоски и камни. Это работает переменная `gameSpeed`.

## Задания для самостоятельной работы

1.  **Три полосы:** В методе `createRoadLines` сделай так, чтобы полоски появлялись не в одной линии по центру, а в двух линиях (разделяя дорогу на 3 полосы).
2.  **Лужи масла:** Нарисуй синее пятно (`oil.png`). Создай класс `Oil`, похожий на `Stone`. Если игрок наезжает на масло — его не убивает, но его машину резко "заносит" (изменяется X координата) или управление меняется на инвертированное (нажал влево — едет вправо) на 2 секунды.
3.  **Рестарт:** Сделай так, чтобы при аварии игра не зависала, а сбрасывала `gameSpeed` обратно на 300 и очищала списки камней, начиная гонку заново.
