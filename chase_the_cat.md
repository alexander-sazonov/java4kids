
# Проект: «Поймай кота» (Catch the Cat)

**Цель:** Создать игру, где ты управляешь ловцом (курсором), а кот бегает по экрану. Кот не стоит на месте: он выбирает случайную точку, бежит туда, потом выбирает новую. Твоя задача — догнать его. Каждый раз, когда ты ловишь кота, он становится быстрее!

**Чему мы научимся:**
1.  Создавать «умное» движение персонажа к цели.
2.  Использовать простую векторную математику (без паники, это всего 3 строчки!).
3.  Повышать сложность игры прямо во время процесса.

---

## Подготовка

1.  Создай новый проект LibGDX.
2.  Найди в интернете (например, на сайте *itch.io* в разделе free assets или просто в Google Картинках по запросу "pixel art cat") 2 картинки:
    *   `cat.png` — Кот (примерно 64x64 пикселя).
    *   `catcher.png` — Ловушка, сачок или просто рука (тоже примерно 64x64). Можно даже взять просто кружок.
3.  Положи их в папку `assets`.

---

## Шаг 1: Родительский класс (GameObject)

Как и всегда, начинаем с фундамента. Этот класс поможет нам не писать один и тот же код дважды.

Создай класс `GameObject`:

```java
import com.badlogic.gdx.graphics.Texture;
import com.badlogic.gdx.graphics.g2d.Batch;
import com.badlogic.gdx.math.Rectangle;

public class GameObject {
    Texture texture;
    float x, y;
    float width, height;
    Rectangle bounds; // Хитбокс

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

## Шаг 2: Кот (Cat) — Самое интересное

Кот должен вести себя "как живой". Он не телепортируется, а бежит. Для этого ему нужно знать:
1.  Куда он хочет попасть прямо сейчас (`targetX`, `targetY`).
2.  Как туда дойти.

Создай класс `Cat`:

```java
import com.badlogic.gdx.Gdx;
import com.badlogic.gdx.graphics.Texture;
import com.badlogic.gdx.math.MathUtils;
import com.badlogic.gdx.math.Vector2;

public class Cat extends GameObject {

    float speed = 200; // Начальная скорость кота
    Vector2 target;    // Точка, куда кот бежит прямо сейчас

    public Cat(Texture texture) {
        super(texture, 100, 100);
        target = new Vector2();
        pickNewTarget(); // Сразу выбираем, куда бежать
    }

    public void update() {
        float dt = Gdx.graphics.getDeltaTime();

        // 1. Вычисляем расстояние до цели
        // (target.x - x) — это сколько осталось пройти по горизонтали
        float dx = target.x - x;
        float dy = target.y - y;

        // Теорема Пифагора, чтобы найти длину пути
        float distance = (float)Math.sqrt(dx*dx + dy*dy);

        // 2. Если мы еще не дошли (расстояние больше 5 пикселей)
        if (distance > 5) {
            // Магия математики: нормализация вектора
            // Мы делим dx на distance, чтобы узнать направление (от -1 до 1)
            // И умножаем на скорость
            x += (dx / distance) * speed * dt;
            y += (dy / distance) * speed * dt;
        } else {
            // Если дошли — выбираем новую цель
            pickNewTarget();
        }

        updateBounds();
    }

    // Кот выбирает случайную точку на экране
    public void pickNewTarget() {
        target.x = MathUtils.random(0, Gdx.graphics.getWidth() - width);
        target.y = MathUtils.random(0, Gdx.graphics.getHeight() - height);
    }

    // Метод для мгновенного перемещения (когда его поймали)
    public void teleport() {
        x = MathUtils.random(0, Gdx.graphics.getWidth() - width);
        y = MathUtils.random(0, Gdx.graphics.getHeight() - height);
        pickNewTarget(); // И сразу строим новый маршрут
    }

    // Ускорение кота
    public void speedUp() {
        speed += 50; // Кот становится быстрее на 50 пунктов
    }
}
```

---

## Шаг 3: Ловец (Catcher)

Это твой персонаж. Он просто приклеен к мышке.

Создай класс `Catcher`:

```java
import com.badlogic.gdx.Gdx;
import com.badlogic.gdx.graphics.Texture;
import com.badlogic.gdx.math.Vector3;

public class Catcher extends GameObject {

    public Catcher(Texture texture) {
        super(texture, 0, 0);
    }

    // В метод update передаем координаты мыши из игрового мира
    public void update(Vector3 mousePos) {
        // Центруем картинку относительно курсора
        x = mousePos.x - width / 2;
        y = mousePos.y - height / 2;

        updateBounds();
    }
}
```

---

## Шаг 4: Главный файл (MyGdxGame)

Собираем всё вместе. Нам снова нужна камера, чтобы координаты мыши совпадали с игровым миром.

```java
import com.badlogic.gdx.ApplicationAdapter;
import com.badlogic.gdx.Gdx;
import com.badlogic.gdx.graphics.OrthographicCamera;
import com.badlogic.gdx.graphics.Texture;
import com.badlogic.gdx.graphics.g2d.SpriteBatch;
import com.badlogic.gdx.math.Vector3;
import com.badlogic.gdx.utils.ScreenUtils;

public class MyGdxGame extends ApplicationAdapter {
    SpriteBatch batch;
    OrthographicCamera camera;

    Texture imgCat, imgCatcher;

    Cat cat;
    Catcher player;

    int score = 0; // Счетчик пойманных котов

    @Override
    public void create() {
        batch = new SpriteBatch();

        // Камера
        camera = new OrthographicCamera();
        camera.setToOrtho(false, 800, 600); // Размер окна

        imgCat = new Texture("cat.png");
        imgCatcher = new Texture("catcher.png");

        cat = new Cat(imgCat);
        player = new Catcher(imgCatcher);
    }

    @Override
    public void render() {
        ScreenUtils.clear(0.8f, 0.8f, 0.9f, 1); // Светло-голубой фон

        camera.update();
        batch.setProjectionMatrix(camera.combined);

        // --- ЛОГИКА ---

        // 1. Получаем координаты мыши и переводим их в игровой мир
        Vector3 mousePos = new Vector3(Gdx.input.getX(), Gdx.input.getY(), 0);
        camera.unproject(mousePos);

        // 2. Двигаем персонажей
        player.update(mousePos);
        cat.update();

        // 3. Проверка: Поймали кота?
        if (player.getBounds().overlaps(cat.getBounds())) {
            score++;
            System.out.println("Пойман! Счет: " + score);

            cat.teleport(); // Кот появляется в новом месте
            cat.speedUp();  // Кот становится быстрее
        }

        // --- ОТРИСОВКА ---
        batch.begin();
        cat.draw(batch);
        player.draw(batch);
        batch.end();
    }

    @Override
    public void dispose() {
        batch.dispose();
        imgCat.dispose();
        imgCatcher.dispose();
    }
}
```

---

## Как это работает?

Когда ты запустишь игру:
1.  **Векторная математика в классе Cat:** Кот смотрит на свою цель (`target`), вычисляет разницу координат (`dx`, `dy`) и двигается по чуть-чуть в этом направлении каждый кадр. Это создает плавное движение.
2.  **Ускорение:** Каждый раз при столкновении вызывается `cat.speedUp()`. Через 10-15 очков поймать кота станет очень сложно, потому что он будет носиться как угорелый!

## Задания для самостоятельной работы

1.  **Кошачьи жизни:** Сделай так, чтобы кот исчезал, если ты не поймал его за 5 секунд (нужно добавить таймер в класс `Cat`). Если он исчез сам — ты теряешь жизнь.
2.  **Телепорт:** Сделай так, чтобы кот не просто бежал к точке, а с шансом 1% (используй `MathUtils.random(1, 100)`) мгновенно менял свое направление движения на новое. Это сделает его поведение непредсказуемым.
3.  **Звуки:** Найди звук мяуканья (`meow.mp3`). Добавь его в игру и проигрывай каждый раз, когда ловишь кота. (Используй класс `Sound`).
