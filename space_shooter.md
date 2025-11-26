---
layout: default
title: Проект: «Space Shooter»
parent: Уроки Java для детей
nav_order: 2
---


# Проект: «Space Shooter»

**Цель:** Создать игру, где космический корабль летает снизу, стреляет лазерами и уничтожает астероиды.
**Новые концепции:** Наследование, ArrayList, итераторы, таймеры появления врагов.

## Подготовка (Assets)
Нужно нарисовать в Paint или найти в интернете 3 картинки (размер примерно 64x64 пикселя) и положить их в папку `assets`:
1.  `ship.png` (Корабль игрока)
2.  `asteroid.png` (Враг)
3.  `bullet.png` (Пуля/Лазер)

---

## Шаг 1: Создаем Родительский Класс (GameObject)

**Теория:** В нашей игре и Корабль, и Астероид, и Пуля имеют много общего: у них есть координаты (x, y), картинка (текстура) и скорость. Чтобы не писать один и тот же код три раза, мы создадим **Родительский класс**.

Создайте новый java-класс `GameObject`.

```java
import com.badlogic.gdx.graphics.Texture;
import com.badlogic.gdx.graphics.g2d.Batch;
import com.badlogic.gdx.math.Rectangle;

public class GameObject {
    // Общие переменные для всех объектов игры
    Texture texture;
    float x, y;
    float width, height;
    float speed;
    Rectangle bounds; // Прямоугольник для проверки столкновений

    // Конструктор
    public GameObject(Texture texture, float x, float y, float speed) {
        this.texture = texture;
        this.x = x;
        this.y = y;
        this.speed = speed;

        this.width = texture.getWidth();
        this.height = texture.getHeight();

        // Создаем прямоугольник вокруг нашей текстуры
        this.bounds = new Rectangle(x, y, width, height);
    }

    // Метод отрисовки, который требовался в задании
    public void draw(Batch batch) {
        batch.draw(texture, x, y);
    }

    // Обновление прямоугольника (хитбокса) при движении
    public void updateBounds() {
        bounds.setPosition(x, y);
    }

    public Rectangle getBounds() {
        return bounds;
    }
}
```

---

## Шаг 2: Создаем Корабль Игрока (Ship)

**Теория:** Корабль — это `GameObject`, но он умеет управляться клавиатурой. Мы используем слово `extends` (расширяет).

Создайте класс `Ship`.

```java
import com.badlogic.gdx.Gdx;
import com.badlogic.gdx.Input;
import com.badlogic.gdx.graphics.Texture;

public class Ship extends GameObject {

    public Ship(Texture texture) {
        // Вызываем конструктор родителя (GameObject)
        // Ставим корабль по центру снизу, скорость 300 пикселей/сек
        super(texture, Gdx.graphics.getWidth() / 2, 20, 300);
    }

    public void update() {
        float dt = Gdx.graphics.getDeltaTime(); // Время последнего кадра

        // Управление (Стрелки или A/D)
        if (Gdx.input.isKeyPressed(Input.Keys.LEFT)) {
            x -= speed * dt;
        }
        if (Gdx.input.isKeyPressed(Input.Keys.RIGHT)) {
            x += speed * dt;
        }

        // Ограничение, чтобы не улетал за экран
        if (x < 0) x = 0;
        if (x > Gdx.graphics.getWidth() - width) x = Gdx.graphics.getWidth() - width;

        // Не забываем обновлять хитбокс!
        updateBounds();
    }
}
```

---

## Шаг 3: Создаем Пулю (Bullet) и Астероид (Asteroid)

Эти классы очень простые. Пуля летит вверх, Астероид — вниз.

**Класс Bullet:**
```java
import com.badlogic.gdx.Gdx;
import com.badlogic.gdx.graphics.Texture;

public class Bullet extends GameObject {

    public Bullet(Texture texture, float x, float y) {
        super(texture, x, y, 500); // Пуля быстрая
    }

    public void update() {
        y += speed * Gdx.graphics.getDeltaTime(); // Летим вверх
        updateBounds();
    }

    // Проверка, улетела ли пуля за экран
    public boolean isOffScreen() {
        return y > Gdx.graphics.getHeight();
    }
}
```

**Класс Asteroid:**
```java
import com.badlogic.gdx.Gdx;
import com.badlogic.gdx.graphics.Texture;
import com.badlogic.gdx.math.MathUtils;

public class Asteroid extends GameObject {

    public Asteroid(Texture texture) {
        // Появляется сверху в случайном месте по X
        super(texture,
              MathUtils.random(0, Gdx.graphics.getWidth() - texture.getWidth()),
              Gdx.graphics.getHeight(),
              MathUtils.random(100, 250)); // Случайная скорость
    }

    public void update() {
        y -= speed * Gdx.graphics.getDeltaTime(); // Падаем вниз
        updateBounds();
    }
}
```

---

## Шаг 4: Главный класс игры (GameScreen)

Здесь мы собираем всё вместе. Объясните детям, что массивы (`[]`) неудобны, когда мы не знаем, сколько будет пуль. Поэтому мы используем `ArrayList` (Динамический список).

В классе `Game` (или `GameScreen`, где у вас основной цикл):

```java
import com.badlogic.gdx.ApplicationAdapter;
import com.badlogic.gdx.Gdx;
import com.badlogic.gdx.Input;
import com.badlogic.gdx.graphics.Texture;
import com.badlogic.gdx.graphics.g2d.SpriteBatch;
import com.badlogic.gdx.utils.ScreenUtils;
import java.util.ArrayList;
import java.util.Iterator;

public class MyGdxGame extends ApplicationAdapter {
    SpriteBatch batch;

    // Текстуры (грузим один раз, чтобы не забивать память!)
    Texture imgShip, imgBullet, imgAsteroid;

    Ship player;
    ArrayList<Bullet> bullets;
    ArrayList<Asteroid> asteroids;

    float asteroidSpawnTimer = 0; // Таймер для создания врагов

    @Override
    public void create() {
        batch = new SpriteBatch();

        // Загрузка картинок
        imgShip = new Texture("ship.png");
        imgBullet = new Texture("bullet.png");
        imgAsteroid = new Texture("asteroid.png");

        // Создание игрока
        player = new Ship(imgShip);

        // Создание списков
        bullets = new ArrayList<>();
        asteroids = new ArrayList<>();
    }

    @Override
    public void render() {
        float dt = Gdx.graphics.getDeltaTime();
        ScreenUtils.clear(0, 0, 0.2f, 1); // Очистка экрана (темно-синий)

        // --- 1. ЛОГИКА (UPDATE) ---

        // Обновление игрока
        player.update();

        // Стрельба (по нажатию Пробела)
        if (Gdx.input.isKeyJustPressed(Input.Keys.SPACE)) {
            // Пуля вылетает из центра корабля
            bullets.add(new Bullet(imgBullet, player.x + player.width/2 - 10, player.y + player.height));
        }

        // Спавн астероидов (каждую секунду)
        asteroidSpawnTimer += dt;
        if (asteroidSpawnTimer > 1.0f) {
            asteroids.add(new Asteroid(imgAsteroid));
            asteroidSpawnTimer = 0;
        }

        // Обновление пуль
        // (Используем итератор, чтобы можно было безопасно удалять объекты во время перебора)
        Iterator<Bullet> bulletIter = bullets.iterator();
        while (bulletIter.hasNext()) {
            Bullet b = bulletIter.next();
            b.update();
            if (b.isOffScreen()) {
                bulletIter.remove(); // Удаляем пулю, если она улетела
            }
        }

        // Обновление астероидов и ПРОВЕРКА СТОЛКНОВЕНИЙ
        Iterator<Asteroid> asteroidIter = asteroids.iterator();
        while (asteroidIter.hasNext()) {
            Asteroid a = asteroidIter.next();
            a.update();

            // Проверка: Астероид врезался в Игрока?
            if (a.getBounds().overlaps(player.getBounds())) {
                System.out.println("GAME OVER!");
                // Тут можно перезапустить игру: create();
            }

            // Проверка: Астероид улетел вниз?
            if (a.y + a.height < 0) {
                asteroidIter.remove();
                continue; // Переходим к следующему, чтобы не проверять пули для удаленного астероида
            }

            // Проверка: Пуля попала в астероид?
            // Нужен вложенный цикл
            Iterator<Bullet> bIter = bullets.iterator();
            while(bIter.hasNext()) {
                Bullet b = bIter.next();
                if (a.getBounds().overlaps(b.getBounds())) {
                    // Попали!
                    bIter.remove(); // Удаляем пулю
                    asteroidIter.remove(); // Удаляем астероид
                    break; // Выходим из цикла пуль, так как астероид уже уничтожен
                }
            }
        }

        // --- 2. ОТРИСОВКА (DRAW) ---
        batch.begin();

        player.draw(batch); // Метод, который мы создали в GameObject

        for (Bullet b : bullets) {
            b.draw(batch);
        }

        for (Asteroid a : asteroids) {
            a.draw(batch);
        }

        batch.end();
    }

    @Override
    public void dispose() {
        batch.dispose();
        imgShip.dispose();
        imgBullet.dispose();
        imgAsteroid.dispose();
    }
}
```
