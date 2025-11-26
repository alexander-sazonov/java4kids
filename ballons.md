# 🎈 Создание игры "Pop the Balloon" на Java (LibGDX)

В этом уроке мы научимся:
1.  Создавать **умные объекты** (классы), которые сами себя рисуют.
2.  Работать со **списками** (`Array`).
3.  Обрабатывать **клики мышки** и переводить координаты.
4.  Автоматически подстраивать хитбоксы под размер картинки.

---

## 📂 Шаг 1: Подготовка ресурсов

Перед началом создай проект LibGDX. В папку `assets` (или `android/assets`) добавь два файла:
1.  `balloon.png` — изображение шарика (лучше с прозрачным фоном).
2.  `background.jpg` — красивый фон (небо).

---

## 🏗️ Шаг 2: Создаем чертеж шарика (Класс Balloon)

Мы будем писать этот класс **внутри** файла вашей игры (или в отдельном файле, если умеешь), но за пределами основного класса игры.

Наш шарик должен быть самостоятельным: он знает свою скорость, хранит свою картинку и умеет себя рисовать.

```java
import com.badlogic.gdx.graphics.Texture;
import com.badlogic.gdx.graphics.g2d.SpriteBatch;
import com.badlogic.gdx.math.MathUtils;
import com.badlogic.gdx.math.Rectangle;

// Класс-чертеж для шарика
class Balloon {
    Rectangle hitbox; // Невидимый прямоугольник для коллизий
    float speed;      // Скорость полета
    Texture image;    // Картинка шарика

    // Конструктор: вызывается при создании нового шарика
    public Balloon(Texture balloonTexture) {
        this.image = balloonTexture; // Запоминаем переданную картинку
        
        hitbox = new Rectangle();
        
        // ВАЖНО: Берем размеры прямо из картинки!
        // Если ты поменяешь картинку на большую, код сам подстроится.
        hitbox.width = image.getWidth();
        hitbox.height = image.getHeight();
        
        // Случайная позиция по X (в пределах ширины экрана 800 минус ширина шарика)
        hitbox.x = MathUtils.random(0, 800 - hitbox.width); 
        
        // Появляемся сразу под экраном (спрятавшись на свою высоту)
        hitbox.y = -hitbox.height; 
        
        // Случайная скорость (чем больше число, тем быстрее)
        speed = MathUtils.random(200, 400);
    }

    // Метод: Шарик рисует сам себя
    public void draw(SpriteBatch batch) {
        batch.draw(image, hitbox.x, hitbox.y);
    }
}
```

---

## ⚙️ Шаг 3: Настройка главного класса

Открой главный файл (обычно `MyGdxGame.java` или похожее). Нам нужно объявить переменные.

```java
import com.badlogic.gdx.ApplicationAdapter;
import com.badlogic.gdx.Gdx;
import com.badlogic.gdx.graphics.OrthographicCamera;
import com.badlogic.gdx.graphics.Texture;
import com.badlogic.gdx.graphics.g2d.SpriteBatch;
import com.badlogic.gdx.math.Vector3;
import com.badlogic.gdx.utils.Array;
import com.badlogic.gdx.utils.ScreenUtils;
import com.badlogic.gdx.utils.TimeUtils;
import java.util.Iterator;

public class BalloonGame extends ApplicationAdapter {
    SpriteBatch batch;
    
    Texture balloonImg; // Картинка шарика (загрузим 1 раз)
    Texture backImg;    // Картинка фона
    
    Array<Balloon> balloons; // Список активных шариков
    
    long lastDropTime; // Время последнего появления шарика
    OrthographicCamera camera; // Камера
    int score; // Счет игрока

    @Override
    public void create() {
        batch = new SpriteBatch();
        
        // 1. Загружаем ресурсы
        balloonImg = new Texture("balloon.png");
        backImg = new Texture("background.jpg");
        
        // 2. Настраиваем камеру (размер окна 800x480)
        camera = new OrthographicCamera();
        camera.setToOrtho(false, 800, 480);
        
        // 3. Инициализируем список и создаем первый шарик
        balloons = new Array<Balloon>();
        spawnBalloon();
        
        score = 0;
    }
    
    // Вспомогательный метод для создания шарика
    private void spawnBalloon() {
        // Передаем нашу загруженную картинку в новый объект
        Balloon b = new Balloon(balloonImg);
        balloons.add(b);
        lastDropTime = TimeUtils.nanoTime();
    }

    // ... методы render и dispose будут ниже ...
}
```

---

## 🎮 Шаг 4: Игровой цикл (Render)

Самое главное происходит здесь. Код выполняется 60 раз в секунду.

```java
    @Override
    public void render() {
        // 1. Очистка экрана
        ScreenUtils.clear(0, 0, 0.2f, 1);
        
        // 2. Обновление камеры
        camera.update();
        batch.setProjectionMatrix(camera.combined);

        // 3. Рисование
        batch.begin();
        batch.draw(backImg, 0, 0); // Рисуем фон
        
        // Просим каждый шарик нарисоваться
        for (Balloon balloon : balloons) {
            balloon.draw(batch);
        }
        // (Здесь можно добавить код для вывода текста со счетом)
        batch.end();

        // --- ЛОГИКА ИГРЫ ---

        // 4. Проверка клика (Лопаем шарики)
        if (Gdx.input.justTouched()) {
            // Получаем координаты клика
            Vector3 touchPos = new Vector3();
            touchPos.set(Gdx.input.getX(), Gdx.input.getY(), 0);
            
            // ВАЖНО: Переводим координаты экрана в координаты игрового мира
            camera.unproject(touchPos);
            
            // Проверяем все шарики
            Iterator<Balloon> iter = balloons.iterator();
            while (iter.hasNext()) {
                Balloon b = iter.next();
                if (b.hitbox.contains(touchPos.x, touchPos.y)) {
                    score++; // Плюс очко
                    // Тут можно добавить звук: popSound.play();
                    iter.remove(); // Удаляем шарик из списка
                }
            }
        }

        // 5. Движение шариков и спавн новых
        // Спавн каждую секунду (1 млрд наносекунд)
        if (TimeUtils.nanoTime() - lastDropTime > 1000000000) {
            spawnBalloon();
        }

        // Обновляем позицию шариков
        Iterator<Balloon> iter = balloons.iterator();
        while (iter.hasNext()) {
            Balloon b = iter.next();
            
            // Двигаем вверх: S = V * T (скорость * время последнего кадра)
            b.hitbox.y += b.speed * Gdx.graphics.getDeltaTime();
            
            // Если улетел за верхний край (480)
            if (b.hitbox.y > 480) {
                iter.remove(); // Просто удаляем
                // Тут можно добавить: lives--; (отнимаем жизнь)
            }
        }
    }
```

---

## 🧹 Шаг 5: Уборка (Dispose)

Не забываем чистить память при выходе из игры.

```java
    @Override
    public void dispose() {
        batch.dispose();
        balloonImg.dispose();
        backImg.dispose();
    }
```

---

## 🏆 Полный код для проверки

Если запутался, вот как должен выглядеть файл целиком:

<details> 
  <summary>Нажми, чтобы развернуть полный код</summary>

```java
package com.mygdx.game; // Твой пакет может отличаться

import com.badlogic.gdx.ApplicationAdapter;
import com.badlogic.gdx.Gdx;
import com.badlogic.gdx.graphics.OrthographicCamera;
import com.badlogic.gdx.graphics.Texture;
import com.badlogic.gdx.graphics.g2d.SpriteBatch;
import com.badlogic.gdx.math.MathUtils;
import com.badlogic.gdx.math.Rectangle;
import com.badlogic.gdx.math.Vector3;
import com.badlogic.gdx.utils.Array;
import com.badlogic.gdx.utils.ScreenUtils;
import com.badlogic.gdx.utils.TimeUtils;
import java.util.Iterator;

// --- КЛАСС ШАРИК ---
class Balloon {
    Rectangle hitbox;
    float speed;
    Texture image;

    public Balloon(Texture balloonTexture) {
        this.image = balloonTexture;
        hitbox = new Rectangle();
        hitbox.width = image.getWidth();
        hitbox.height = image.getHeight();
        hitbox.x = MathUtils.random(0, 800 - hitbox.width);
        hitbox.y = -hitbox.height;
        speed = MathUtils.random(200, 400);
    }

    public void draw(SpriteBatch batch) {
        batch.draw(image, hitbox.x, hitbox.y);
    }
}

// --- ГЛАВНЫЙ КЛАСС ---
public class BalloonGame extends ApplicationAdapter {
    SpriteBatch batch;
    Texture balloonImg;
    Texture backImg;
    Array<Balloon> balloons;
    OrthographicCamera camera;
    long lastDropTime;
    int score;

    @Override
    public void create() {
        batch = new SpriteBatch();
        balloonImg = new Texture("balloon.png");
        backImg = new Texture("background.jpg");
        camera = new OrthographicCamera();
        camera.setToOrtho(false, 800, 480);
        balloons = new Array<>();
        spawnBalloon();
        score = 0;
    }

    private void spawnBalloon() {
        Balloon b = new Balloon(balloonImg);
        balloons.add(b);
        lastDropTime = TimeUtils.nanoTime();
    }

    @Override
    public void render() {
        ScreenUtils.clear(0, 0, 0.2f, 1);
        camera.update();
        batch.setProjectionMatrix(camera.combined);

        batch.begin();
        batch.draw(backImg, 0, 0);
        for (Balloon b : balloons) {
            b.draw(batch);
        }
        batch.end();

        if (Gdx.input.justTouched()) {
            Vector3 touchPos = new Vector3();
            touchPos.set(Gdx.input.getX(), Gdx.input.getY(), 0);
            camera.unproject(touchPos);

            Iterator<Balloon> iter = balloons.iterator();
            while (iter.hasNext()) {
                Balloon b = iter.next();
                if (b.hitbox.contains(touchPos.x, touchPos.y)) {
                    score++;
                    System.out.println("Score: " + score); // Вывод в консоль
                    iter.remove();
                }
            }
        }

        if (TimeUtils.nanoTime() - lastDropTime > 1000000000) {
            spawnBalloon();
        }

        Iterator<Balloon> iter = balloons.iterator();
        while (iter.hasNext()) {
            Balloon b = iter.next();
            b.hitbox.y += b.speed * Gdx.graphics.getDeltaTime();
            if (b.hitbox.y > 480) {
                iter.remove();
            }
        }
    }

    @Override
    public void dispose() {
        batch.dispose();
        balloonImg.dispose();
        backImg.dispose();
    }
}
```
</details>

---

## 🔥 Задания для самостоятельной работы

Ты уже написал основу. Теперь преврати это в настоящую игру!

1.  **Система Жизней:**
    *   Создай переменную `int lives = 3;`.
    *   Если шарик улетает за верх экрана (`y > 480`), уменьшай жизни.
    *   Если `lives <= 0`, показывай картинку "Game Over".
2.  **Ускорение:**
    *   Сделай так, чтобы при каждом создании нового шарика `minSpeed` и `maxSpeed` немного увеличивались. Игра должна становиться сложнее!
3.  **Бонус:**
    *   Найди картинку бомбы. Сделай так, чтобы с шансом 10% (используй `MathUtils.random(1, 10) == 1`) спавнилась бомба. Если кликнуть по ней — конец игры.
