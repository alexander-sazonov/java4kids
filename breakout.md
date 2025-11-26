
# Проект: «Арканоид» (Breakout)

**Цель:** Создать классическую игру. Сверху — стена из кирпичей. Снизу — платформа (ракетка), которой ты управляешь мышкой. По экрану летает мяч, отскакивает от стен и разбивает кирпичи. Если мяч упадет вниз — проигрыш.

**Чему мы научимся:**
1.  Физике отскока (отражение скорости).
2.  Созданию сложных уровней с помощью вложенных циклов.
3.  Обработке столкновений между движущимся и статичным объектами.

---

## Подготовка

Создай новый проект LibGDX.
Нарисуй или скачай 3 картинки и положи их в папку `assets`:
1.  `paddle.png` — Ракетка (длинный прямоугольник, например 100x20 пикселей).
2.  `ball.png` — Мячик (квадрат или круг, 20x20 пикселей).
3.  `brick.png` — Кирпич (прямоугольник, например 60x30 пикселей).

---

## Шаг 1: Родительский класс (GameObject)

Наш стандартный класс, который мы используем во всех проектах. Он хранит картинку, координаты и умеет рисовать себя.

Создай класс `GameObject`:

```java
import com.badlogic.gdx.graphics.Texture;
import com.badlogic.gdx.graphics.g2d.Batch;
import com.badlogic.gdx.math.Rectangle;

public class GameObject {
    Texture texture;
    float x, y;
    float width, height;
    Rectangle bounds; // Прямоугольник для столкновений

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

## Шаг 2: Ракетка (Paddle)

Ракетка будет двигаться за мышкой, но только по горизонтали (ось X). По вертикали она закреплена внизу.

Создай класс `Paddle`:

```java
import com.badlogic.gdx.Gdx;
import com.badlogic.gdx.graphics.Texture;

public class Paddle extends GameObject {

    public Paddle(Texture texture) {
        // Ставим ракетку по центру снизу (Y = 20)
        super(texture, Gdx.graphics.getWidth() / 2, 20);
    }

    public void update() {
        // Получаем координату мышки по X
        // Мы хотим, чтобы центр ракетки был там, где курсор
        x = Gdx.input.getX() - width / 2;

        // Не даем ракетке уехать за границы экрана
        if (x < 0) x = 0;
        if (x > Gdx.graphics.getWidth() - width) x = Gdx.graphics.getWidth() - width;

        updateBounds();
    }
}
```

---

## Шаг 3: Мяч (Ball)

Самый сложный объект. Мяч имеет скорость по X и скорость по Y. Когда он врезается в стену, он должен "отзеркалить" свою скорость.

Создай класс `Ball`:

```java
import com.badlogic.gdx.Gdx;
import com.badlogic.gdx.graphics.Texture;

public class Ball extends GameObject {

    float speedX = 200; // Скорость по горизонтали
    float speedY = 200; // Скорость по вертикали

    public Ball(Texture texture) {
        super(texture, Gdx.graphics.getWidth() / 2, 200); // Начинаем с середины
    }

    public void update() {
        float dt = Gdx.graphics.getDeltaTime();
        
        // Двигаем мяч
        x += speedX * dt;
        y += speedY * dt;

        // Отскок от стен (левая и правая)
        if (x < 0 || x > Gdx.graphics.getWidth() - width) {
            speedX = -speedX; // Меням направление влево/вправо
        }

        // Отскок от потолка
        if (y > Gdx.graphics.getHeight() - height) {
            speedY = -speedY; // Меняем направление вниз
        }

        updateBounds();
    }
    
    // Методы, чтобы заставить мяч отскочить вручную (от ракетки или кирпича)
    public void reverseY() {
        speedY = -speedY;
    }
    
    // Проверка: упал ли мяч вниз?
    public boolean isLost() {
        return y < 0;
    }
}
```

---

## Шаг 4: Кирпич (Brick)

Кирпич — это просто неподвижный объект.

Создай класс `Brick`:

```java
import com.badlogic.gdx.graphics.Texture;

public class Brick extends GameObject {
    public Brick(Texture texture, float x, float y) {
        super(texture, x, y);
    }
}
```

---

## Шаг 5: Сборка игры (MyGdxGame)

Здесь мы создадим стену из кирпичей, используя двойной цикл, и настроим проверку столкновений.

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
    
    Texture imgPaddle, imgBall, imgBrick;
    
    Paddle paddle;
    Ball ball;
    ArrayList<Brick> bricks;

    @Override
    public void create() {
        batch = new SpriteBatch();
        
        imgPaddle = new Texture("paddle.png");
        imgBall = new Texture("ball.png");
        imgBrick = new Texture("brick.png");
        
        paddle = new Paddle(imgPaddle);
        ball = new Ball(imgBall);
        bricks = new ArrayList<>();
        
        createLevel();
    }
    
    // Создаем сетку кирпичей
    private void createLevel() {
        int rows = 5;  // Сколько рядов кирпичей
        int cols = 8;  // Сколько кирпичей в ряду
        
        // Отступы, чтобы кирпичи были по центру
        float startX = 50;
        float startY = Gdx.graphics.getHeight() - 100;
        
        for (int row = 0; row < rows; row++) {
            for (int col = 0; col < cols; col++) {
                // Координаты: col умножаем на ширину кирпича + зазор 5 пикселей
                float x = startX + col * (imgBrick.getWidth() + 5);
                float y = startY - row * (imgBrick.getHeight() + 5);
                
                bricks.add(new Brick(imgBrick, x, y));
            }
        }
    }

    @Override
    public void render() {
        ScreenUtils.clear(0.2f, 0.2f, 0.2f, 1); // Темно-серый фон
        
        // --- 1. ЛОГИКА ---
        
        paddle.update();
        ball.update();
        
        // Проверка: Мяч упал?
        if (ball.isLost()) {
            System.out.println("GAME OVER!");
            // Можно перезапустить мяч:
            ball = new Ball(imgBall); 
        }
        
        // Столкновение Мяча с Ракеткой
        if (ball.getBounds().overlaps(paddle.getBounds())) {
            // Мяч летит вниз И касается ракетки
            // Дополнительная проверка "ball.y > paddle.y" нужна, 
            // чтобы мяч не прилипал к ракетке сбоку
            if (ball.y > paddle.y) {
                ball.reverseY(); // Отбиваем мяч вверх
                
                // Хитрый трюк: немного поднимаем мяч, чтобы он не застрял внутри ракетки
                ball.y = paddle.y + paddle.height + 1;
                ball.updateBounds();
            }
        }
        
        // Столкновение Мяча с Кирпичами
        Iterator<Brick> iter = bricks.iterator();
        while (iter.hasNext()) {
            Brick b = iter.next();
            
            if (ball.getBounds().overlaps(b.getBounds())) {
                iter.remove();  // Удаляем кирпич
                ball.reverseY(); // Мяч отскакивает
                break; // Важно! За один кадр разбиваем только один кирпич
            }
        }
        
        // Победа?
        if (bricks.isEmpty()) {
            System.out.println("YOU WIN!");
            createLevel(); // Создаем новый уровень
        }

        // --- 2. ОТРИСОВКА ---
        batch.begin();
        
        paddle.draw(batch);
        ball.draw(batch);
        
        for (Brick b : bricks) {
            b.draw(batch);
        }
        
        batch.end();
    }

    @Override
    public void dispose() {
        batch.dispose();
        imgPaddle.dispose();
        imgBall.dispose();
        imgBrick.dispose();
    }
}
```

---

## Запуск игры

1.  Запусти проект.
2.  Ты увидишь ряды кирпичей сверху.
3.  Управляй ракеткой мышкой, чтобы не дать мячу упасть.
4.  Когда мяч разбивает кирпич, тот исчезает, а мяч отлетает обратно.

## Задания для самостоятельной работы

1.  **Ускорение:** В классе `Ball` создай метод `increaseSpeed()`, который умножает скорость на 1.1 (на 10%). Вызывай этот метод каждый раз, когда мяч касается ракетки. Игра станет сложнее!
2.  **Разноцветные кирпичи:** В классе `Brick` добавь поле `color`. В `createLevel` делай так: 1-й ряд — красные кирпичи, 2-й — желтые и т.д. (Для этого нужно использовать метод `batch.setColor()` перед отрисовкой кирпича и `batch.setColor(Color.WHITE)` после).
3.  **"Умный" отскок:** Сейчас мяч всегда отскакивает под одним углом. Попробуй сделать так: если мяч ударился о левую часть ракетки — он летит влево, если о правую — вправо. 
    *Подсказка:* Нужно менять `speedX` в зависимости от того, где именно ударился мяч (`ball.x - paddle.x`).
