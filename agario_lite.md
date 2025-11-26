
# Проект: «Голодный Шарик» (Agario Lite)

**Цель:** Ты управляешь шариком. На экране появляется еда. Твоя задача — съедать её. С каждым съеденным кусочком твой персонаж становится больше.

**Чему научимся:**
1.  **Круглые коллизии:** Мы научимся проверять столкновения кругов, а не квадратов.
2.  **Масштабирование:** Мы научимся рисовать картинки так, чтобы они меняли размер, но не теряли качество (не становились размытыми).

---

## Подготовка

1.  Создай новый проект LibGDX.
2.  **ВАЖНО ПРО КАРТИНКИ:** Так как наш шар будет расти, нам нужно изображение высокого качества. В программировании игр есть правило: *"Лучше взять большую картинку и уменьшить её, чем взять маленькую и растянуть"*.
    *   `player.png` — Найди **большое** изображение круга (например, **256x256** или **512x512** пикселей).
    *   `food.png` — Еда (можно поменьше, например 64x64).
    *   Убедись, что картинки имеют **прозрачный фон**.
3.  Положи их в папку `assets`.

---

## Шаг 1: Родительский класс (GameObject)

Вместо `Rectangle` мы будем использовать класс `Circle` (Круг). Также мы научим наш объект менять размер.

Создай класс `GameObject`:

```java
import com.badlogic.gdx.graphics.Texture;
import com.badlogic.gdx.graphics.g2d.Batch;
import com.badlogic.gdx.math.Circle;

public class GameObject {
    Texture texture;
    float x, y;          // Координаты для рисования (нижний левый угол картинки)
    float size;          // Текущий размер (диаметр) объекта
    Circle bounds;       // Круг для проверки столкновений

    public GameObject(Texture texture, float x, float y, float size) {
        this.texture = texture;
        this.x = x;
        this.y = y;
        this.size = size;
        
        // Создаем математический круг. 
        // Ему нужно передать координаты ЦЕНТРА и РАДИУС.
        // Центр = координата + половина размера. Радиус = половина размера.
        this.bounds = new Circle(x + size / 2, y + size / 2, size / 2);
    }

    public void draw(Batch batch) {
        // Рисуем картинку, указывая ей конкретный размер (size)
        // SpriteBatch сам сожмет большую картинку до этого размера
        batch.draw(texture, x, y, size, size);
    }

    // Обновляем позицию круга-хитбокса при движении
    public void updateBounds() {
        float radius = size / 2;
        bounds.set(x + radius, y + radius, radius);
    }

    public Circle getBounds() {
        return bounds;
    }
}
```

---

## Шаг 2: Еда (Food)

Еда — это маленькие шарики. Они появляются в случайных местах.

Создай класс `Food`:

```java
import com.badlogic.gdx.Gdx;
import com.badlogic.gdx.graphics.Texture;
import com.badlogic.gdx.math.MathUtils;

public class Food extends GameObject {

    public Food(Texture texture) {
        // Создаем еду размером 30x30 пикселей.
        // Координаты пока ставим 0,0, так как сразу вызовем respawn()
        super(texture, 0, 0, 30); 
        respawn(); 
    }

    // Метод для переноса еды в новое место
    public void respawn() {
        x = MathUtils.random(0, Gdx.graphics.getWidth() - size);
        y = MathUtils.random(0, Gdx.graphics.getHeight() - size);
        updateBounds();
    }
}
```

---

## Шаг 3: Игрок (Player)

Игрок начинает маленьким, но умеет расти.

Создай класс `Player`:

```java
import com.badlogic.gdx.Gdx;
import com.badlogic.gdx.Input;
import com.badlogic.gdx.graphics.Texture;

public class Player extends GameObject {
    
    float speed = 200; // Скорость движения

    public Player(Texture texture) {
        // На старте мы хотим быть маленькими (например, 64 пикселя),
        // даже если наша картинка огромная (512 пикселей).
        super(texture, Gdx.graphics.getWidth() / 2, Gdx.graphics.getHeight() / 2, 64);
    }

    public void update() {
        float dt = Gdx.graphics.getDeltaTime();

        // Управление стрелками
        if (Gdx.input.isKeyPressed(Input.Keys.LEFT))  x -= speed * dt;
        if (Gdx.input.isKeyPressed(Input.Keys.RIGHT)) x += speed * dt;
        if (Gdx.input.isKeyPressed(Input.Keys.UP))    y += speed * dt;
        if (Gdx.input.isKeyPressed(Input.Keys.DOWN))  y -= speed * dt;

        // Ограничение экрана
        if (x < 0) x = 0;
        if (y < 0) y = 0;
        if (x > Gdx.graphics.getWidth() - size) x = Gdx.graphics.getWidth() - size;
        if (y > Gdx.graphics.getHeight() - size) y = Gdx.graphics.getHeight() - size;

        updateBounds();
    }

    // Самый главный метод - РОСТ
    public void grow() {
        // Увеличиваем размер на 5 пикселей
        size += 5;
        
        // Немного сдвигаем координаты влево и вниз, 
        // чтобы казалось, что мы растем из центра
        x -= 2.5f;
        y -= 2.5f;
        
        // Физика: чем больше объект, тем тяжелее ему двигаться
        if (speed > 50) {
            speed -= 2; 
        }
        
        updateBounds();
    }
}
```

---

## Шаг 4: Главный файл (MyGdxGame)

Здесь мы применим "Магию сглаживания" (Texture Filtering), чтобы наши красивые большие картинки выглядели хорошо при любом размере.

```java
import com.badlogic.gdx.ApplicationAdapter;
import com.badlogic.gdx.Gdx;
import com.badlogic.gdx.graphics.Texture;
import com.badlogic.gdx.graphics.g2d.SpriteBatch;
import com.badlogic.gdx.utils.ScreenUtils;
import java.util.ArrayList;

public class MyGdxGame extends ApplicationAdapter {
    SpriteBatch batch;
    
    Texture imgPlayer, imgFood;
    
    Player player;
    ArrayList<Food> foods;
    
    int score = 0;

    @Override
    public void create() {
        batch = new SpriteBatch();
        
        imgPlayer = new Texture("player.png");
        imgFood = new Texture("food.png");

        // --- НАСТРОЙКА КАЧЕСТВА ---
        // Linear Filter (Линейная фильтрация) сглаживает пиксели.
        // Благодаря этому большая картинка при уменьшении не рябит,
        // а при увеличении не распадается на квадраты.
        imgPlayer.setFilter(Texture.TextureFilter.Linear, Texture.TextureFilter.Linear);
        imgFood.setFilter(Texture.TextureFilter.Linear, Texture.TextureFilter.Linear);
        // --------------------------
        
        player = new Player(imgPlayer);
        
        // Создаем 20 кусочков еды
        foods = new ArrayList<>();
        for (int i = 0; i < 20; i++) {
            foods.add(new Food(imgFood));
        }
    }

    @Override
    public void render() {
        // Светло-серый фон
        ScreenUtils.clear(0.9f, 0.9f, 0.9f, 1);
        
        // --- ЛОГИКА ---
        player.update();
        
        for (Food f : foods) {
            // Проверка столкновения КРУГОВ (overlaps)
            if (player.getBounds().overlaps(f.getBounds())) {
                f.respawn();   // Еда убегает
                player.grow(); // Игрок растет
                score++;
                System.out.println("Ням! Текущий размер: " + (int)player.size);
            }
        }

        // --- ОТРИСОВКА ---
        batch.begin();
        
        // Сначала рисуем еду
        for (Food f : foods) {
            f.draw(batch);
        }
        
        // Потом игрока (чтобы он перекрывал еду)
        player.draw(batch);
        
        batch.end();
    }

    @Override
    public void dispose() {
        batch.dispose();
        imgPlayer.dispose();
        imgFood.dispose();
    }
}
```

---

## Почему это работает лучше?

1.  **Качество:** Мы взяли картинку 512px. Когда мы создали игрока, мы сказали ему быть размером 64px. Видеокарта "сжала" картинку. Когда игрок съедает еду, он становится 70px, 80px и так далее. Видеокарта просто "распрямляет" сжатую картинку обратно. Она всегда остается четкой!
2.  **Фильтр (setFilter):** Без этой команды, при изменении размера, края круга могли бы стать "лесенкой" (зубчатыми). Фильтр делает их гладкими.

## Задания для самостоятельной работы

1.  **Лимит роста:** Добавь проверку в метод `grow`. Если `size` больше 300, игрок перестает расти (иначе он закроет весь экран!).
2.  **Опасные шипы:** Найди картинку колючки (`spike.png`). Создай класс `Spike`.
    *   Если игрок наезжает на шип, его `size` уменьшается сразу на 20 пикселей (он "сдувается"), а шип исчезает.
3.  **Победа:** Если игрок набрал 50 очков, выведи в консоль сообщение о победе и останови игру (можно сделать `speed = 0`).
