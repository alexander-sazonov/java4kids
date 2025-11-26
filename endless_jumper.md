
# Проект: «Прыг-скок» (Endless Jumper)

**Цель:** Создать игру-раннер. Твой персонаж бежит слева направо (на самом деле он стоит на месте, а мир движется ему навстречу). На пути встречаются препятствия. Твоя задача — вовремя нажимать на экран, чтобы перепрыгивать их.

**Главная фишка:** Мы не будем использовать сложную гравитацию. Мы сделаем красивый прыжок с помощью математики (синусоиды), чтобы персонаж двигался по идеальной дуге.

---

## Подготовка

1.  Создай новый проект LibGDX.
2.  Скачай бесплатные ресурсы (например, с сайта *kenney.nl* или *itch.io* по запросу "2d platformer assets").
3.  Нам нужны 2 картинки в папке `assets`:
    *   `runner.png` — Персонаж (например, ниндзя или динозаврик, 64x64).
    *   `cactus.png` — Препятствие (коробка, камень или кактус, 50x50).

---

## Шаг 1: Родительский класс (GameObject)

Наша стандартная основа для всех объектов.

Создай класс `GameObject`:

```java
import com.badlogic.gdx.graphics.Texture;
import com.badlogic.gdx.graphics.g2d.Batch;
import com.badlogic.gdx.math.Rectangle;

public class GameObject {
    Texture texture;
    float x, y;
    float width, height;
    Rectangle bounds; 

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

    public void updateBounds() {
        bounds.setPosition(x, y);
    }
    
    public Rectangle getBounds() {
        return bounds;
    }
}
```

---

## Шаг 2: Бегун (Runner) — Логика прыжка

Здесь мы используем **Синусоиду** (`Math.sin`).
*   Синус — это волна. Она плавно идет вверх от 0 до 1, а потом плавно вниз до 0.
*   Это идеально подходит для прыжка, если мы не хотим мучиться с гравитацией!

Создай класс `Runner`:

```java
import com.badlogic.gdx.Gdx;
import com.badlogic.gdx.graphics.Texture;
import com.badlogic.gdx.math.MathUtils;

public class Runner extends GameObject {
    
    float groundY;       // Уровень земли (где ноги персонажа)
    boolean isJumping = false;
    float jumpTimer = 0; // Время полета
    
    float jumpHeight = 150; // Насколько высоко прыгаем (в пикселях)
    float jumpSpeed = 3.0f; // Как быстро происходит прыжок

    public Runner(Texture texture) {
        super(texture, 100, 100); // Игрок всегда стоит на X=100
        groundY = 100;            // Запоминаем уровень земли
    }

    public void update() {
        // Если нажали мышкой (или коснулись экрана) и мы НЕ в воздухе
        if (Gdx.input.justTouched() && !isJumping) {
            isJumping = true;
            jumpTimer = 0; // Начинаем отсчет прыжка
        }

        // Если мы в процессе прыжка
        if (isJumping) {
            float dt = Gdx.graphics.getDeltaTime();
            jumpTimer += dt * jumpSpeed;

            // Магия математики:
            // MathUtils.sin(0) = 0 (начало)
            // MathUtils.sin(PI / 2) = 1 (верхняя точка)
            // MathUtils.sin(PI) = 0 (приземление)
            
            // Мы прибавляем к уровню земли высоту прыжка умноженную на синус
            y = groundY + MathUtils.sin(jumpTimer) * jumpHeight;

            // Если таймер больше числа Пи (3.14), значит волна закончилась
            if (jumpTimer > MathUtils.PI) {
                y = groundY;      // Возвращаем точно на землю
                isJumping = false; // Прыжок окончен
            }
        }
        
        updateBounds();
    }
}
```

---

## Шаг 3: Препятствие (Obstacle)

Препятствия просто движутся влево. Когда они уходят за экран, мы их удаляем, чтобы не засорять память.

Создай класс `Obstacle`:

```java
import com.badlogic.gdx.Gdx;
import com.badlogic.gdx.graphics.Texture;

public class Obstacle extends GameObject {
    
    public Obstacle(Texture texture, float startX, float groundY) {
        super(texture, startX, groundY);
    }

    // gameSpeed — это скорость, с которой бежит мир
    public void update(float gameSpeed) {
        float dt = Gdx.graphics.getDeltaTime();
        
        x -= gameSpeed * dt; // Двигаемся влево
        
        updateBounds();
    }

    // Проверка: ушли ли мы за левый край экрана?
    public boolean isOffScreen() {
        return x + width < 0;
    }
}
```

---

## Шаг 4: Сборка игры (MyGdxGame)

Здесь мы создаем иллюзию бесконечного бега. Мы будем создавать препятствия справа за краем экрана, они будут проезжать мимо игрока и удаляться.

```java
import com.badlogic.gdx.ApplicationAdapter;
import com.badlogic.gdx.Gdx;
import com.badlogic.gdx.graphics.Texture;
import com.badlogic.gdx.graphics.g2d.SpriteBatch;
import com.badlogic.gdx.math.MathUtils;
import com.badlogic.gdx.utils.ScreenUtils;
import java.util.ArrayList;
import java.util.Iterator;

public class MyGdxGame extends ApplicationAdapter {
    SpriteBatch batch;
    
    Texture imgRunner, imgCactus;
    
    Runner player;
    ArrayList<Obstacle> obstacles;
    
    float gameSpeed = 300;     // Скорость бега (пикселей в секунду)
    float spawnTimer = 0;      // Таймер создания препятствий
    int score = 0;             // Очки

    @Override
    public void create() {
        batch = new SpriteBatch();
        
        imgRunner = new Texture("runner.png");
        imgCactus = new Texture("cactus.png");
        
        player = new Runner(imgRunner);
        obstacles = new ArrayList<>();
    }

    @Override
    public void render() {
        ScreenUtils.clear(0.5f, 0.8f, 1, 1); // Небесно-голубой фон
        
        float dt = Gdx.graphics.getDeltaTime();

        // --- ЛОГИКА ---
        
        player.update();
        
        // Генерация препятствий
        spawnTimer += dt;
        // Каждые 1.5 - 2.5 секунды создаем новый кактус
        // (чем больше gameSpeed, тем чаще можно создавать, но пока сделаем просто по времени)
        if (spawnTimer > 2.0f) {
            // Создаем кактус далеко справа (ширина экрана + 50)
            obstacles.add(new Obstacle(imgCactus, Gdx.graphics.getWidth() + 50, 100));
            spawnTimer = 0;
            
            // Немного случайности, чтобы не было скучно:
            // следующий кактус появится через случайное время
            spawnTimer -= MathUtils.random(0.0f, 1.0f);
        }

        // Обновление препятствий
        Iterator<Obstacle> iter = obstacles.iterator();
        while (iter.hasNext()) {
            Obstacle obs = iter.next();
            obs.update(gameSpeed);
            
            // Проверка столкновения (Проигрыш)
            if (obs.getBounds().overlaps(player.getBounds())) {
                System.out.println("ОЙ! Споткнулся. Счет: " + score);
                score = 0;          // Сброс очков
                obstacles.clear();  // Удаляем все препятствия
                break;              // Прерываем цикл, чтобы не было ошибок
            }
            
            // Если препятствие ушло за экран
            if (obs.isOffScreen()) {
                iter.remove();
                score++;
                System.out.println("Прыжок! Счет: " + score);
            }
        }

        // --- ОТРИСОВКА ---
        batch.begin();
        
        // Рисуем "Землю" (просто прямоугольник или линию)
        // Для простоты можно пока не рисовать, но представим, что Y=100 это пол.
        
        player.draw(batch);
        
        for (Obstacle obs : obstacles) {
            obs.draw(batch);
        }
        
        batch.end();
    }

    @Override
    public void dispose() {
        batch.dispose();
        imgRunner.dispose();
        imgCactus.dispose();
    }
}
```

---

## Как это работает?

1.  **Runner (Бегун):** Он меняет только свой `Y`. `X` у него фиксированный (он не бежит вперед, он прыгает на месте).
2.  **Obstacle (Кактус):** Он меняет свой `X` (уменьшает), создавая иллюзию, что мы бежим мимо него.
3.  **Синус:** Благодаря строчке `MathUtils.sin(jumpTimer)`, прыжок выглядит очень плавным, как дуга, без использования сложных физических формул.

## Задания для самостоятельной работы

1.  **Ускорение:** В методе `render`, добавь `gameSpeed += 5 * dt;`. Теперь с каждой секундой игра будет становиться быстрее!
2.  **Двойной прыжок:** Попробуй сделать так, чтобы игрок мог прыгнуть еще раз, находясь в воздухе.
    *   *Подсказка:* Добавь переменную `int jumpsLeft = 2`. При клике уменьшай её. Сбрасывай на 2, когда приземлился.
3.  **Фон:** Нарисуй облако (`cloud.png`). Создай для него класс `Cloud`, который тоже летит влево, но медленнее (`gameSpeed / 2`). Это создаст эффект глубины (параллакс).
