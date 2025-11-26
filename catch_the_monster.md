
# Проект: «Охота на Монстров» (Whack-a-Mole)

**Цель:** Мы создадим игру, где на поле расположены норы. Из нор случайно выскакивают монстры. Твоя задача — успеть кликнуть по монстру мышкой, пока он не спрятался.

**Чему мы научимся:**
1.  Работать с мышкой и кликами.
2.  Создавать сетку объектов (ряды и колонки).
3.  Использовать таймеры.

---

## Подготовка

Перед началом создай новый проект LibGDX (так же, как мы делали раньше).
Найди в интернете или нарисуй две картинки и положи их в папку `assets`:
1.  `monster.png` — Картинка монстра (желательно квадратная, например 64x64 или 128x128 пикселей).
2.  `hole.png` — Картинка норы (черный круг или овал), такого же размера.

---

## Шаг 1: Создаем основу (Класс GameObject)

Чтобы не писать один и тот же код для координат и отрисовки много раз, мы создадим главный родительский класс.

1.  Создай новый Java-класс с именем `GameObject`.
2.  Напиши в нем следующий код:

```java
import com.badlogic.gdx.graphics.Texture;
import com.badlogic.gdx.graphics.g2d.Batch;
import com.badlogic.gdx.math.Rectangle;

public class GameObject {
    Texture texture;
    float x, y;
    float width, height;
    Rectangle bounds; // Невидимая рамка вокруг объекта

    // Конструктор: вызывается при создании объекта
    public GameObject(Texture texture, float x, float y) {
        this.texture = texture;
        this.x = x;
        this.y = y;
        this.width = texture.getWidth();
        this.height = texture.getHeight();
        
        // Создаем прямоугольник для обработки кликов
        this.bounds = new Rectangle(x, y, width, height);
    }

    // Метод отрисовки
    public void draw(Batch batch) {
        batch.draw(texture, x, y);
    }
    
    // Метод проверки: попали ли мы в объект мышкой?
    // clickX и clickY — это координаты клика в игровом мире
    public boolean isHit(float clickX, float clickY) {
        return bounds.contains(clickX, clickY);
    }
}
```

---

## Шаг 2: Создаем Монстра (Класс Monster)

Наш монстр — это непростой объект. Он умеет прятаться и появляться.

1.  Создай новый класс `Monster`.
2.  Он должен наследовать `GameObject` (используем слово `extends`).

```java
import com.badlogic.gdx.Gdx;
import com.badlogic.gdx.graphics.Texture;
import com.badlogic.gdx.math.MathUtils;
import com.badlogic.gdx.graphics.g2d.Batch;

public class Monster extends GameObject {
    
    private boolean isVisible = false;   // Виден ли монстр сейчас?
    private float timeVisible = 0;       // Сколько времени он уже виден
    private float maxVisibleTime = 1.0f; // Через сколько секунд исчезнет сам
    
    private Texture holeTexture; // Текстура норы

    public Monster(Texture texture, Texture holeTexture, float x, float y) {
        super(texture, x, y); // Передаем данные в GameObject
        this.holeTexture = holeTexture;
    }

    // Логика жизни монстра: обновляем таймер
    public void update() {
        if (isVisible) {
            float dt = Gdx.graphics.getDeltaTime();
            timeVisible += dt;
            
            // Если время вышло — монстр прячется
            if (timeVisible > maxVisibleTime) {
                hide();
            }
        }
    }

    // Команда "Появись!"
    public void show() {
        isVisible = true;
        timeVisible = 0;
        // Монстр будет виден случайное время: от 0.5 до 1.5 секунд
        maxVisibleTime = MathUtils.random(0.5f, 1.5f); 
    }

    // Команда "Спрячься!"
    public void hide() {
        isVisible = false;
    }
    
    // Геттер: узнать, виден ли монстр
    public boolean isVisible() {
        return isVisible;
    }

    // Переопределяем (заменяем) метод draw
    @Override
    public void draw(Batch batch) {
        // 1. Сначала всегда рисуем нору (фон)
        batch.draw(holeTexture, x, y);

        // 2. Если монстр "вылез", рисуем его поверх норы
        if (isVisible) {
            super.draw(batch); 
        }
    }
}
```

---

## Шаг 3: Собираем игру (Класс MyGdxGame)

Теперь переходим в главный файл `MyGdxGame`. Здесь мы создадим сетку монстров, настроим камеру и будем обрабатывать клики мышкой.

Нам понадобится **Камера**, потому что координаты мышки на экране (считаются сверху вниз) не совпадают с координатами игры (считаются снизу вверх). Камера поможет это исправить.

Вставь этот код в `MyGdxGame.java`:

```java
import com.badlogic.gdx.ApplicationAdapter;
import com.badlogic.gdx.Gdx;
import com.badlogic.gdx.graphics.OrthographicCamera;
import com.badlogic.gdx.graphics.Texture;
import com.badlogic.gdx.graphics.g2d.SpriteBatch;
import com.badlogic.gdx.math.MathUtils;
import com.badlogic.gdx.math.Vector3;
import com.badlogic.gdx.utils.ScreenUtils;
import java.util.ArrayList;

public class MyGdxGame extends ApplicationAdapter {
    SpriteBatch batch;
    OrthographicCamera camera; // Наша камера
    
    Texture imgMonster;
    Texture imgHole;
    
    // Список всех монстров на поле
    ArrayList<Monster> monsters;
    
    float spawnTimer = 0; // Таймер для появления нового монстра
    int score = 0;        // Счёт очков

    @Override
    public void create() {
        batch = new SpriteBatch();
        
        // Настройка камеры. 800x600 — размер нашего игрового окна
        camera = new OrthographicCamera();
        camera.setToOrtho(false, 800, 600);
        
        imgMonster = new Texture("monster.png");
        imgHole = new Texture("hole.png");
        
        monsters = new ArrayList<>();
        
        createGrid(); // Вызываем метод создания сетки
    }
    
    // Метод, который расставит норы ровными рядами
    private void createGrid() {
        float startX = 200; // Отступ слева
        float startY = 150; // Отступ снизу
        float gap = 150;    // Расстояние между норами
        
        // Двойной цикл: 3 ряда по 3 колонки
        for (int row = 0; row < 3; row++) {
            for (int col = 0; col < 3; col++) {
                // Вычисляем координаты для текущей норы
                float x = startX + col * gap;
                float y = startY + row * gap;
                
                monsters.add(new Monster(imgMonster, imgHole, x, y));
            }
        }
    }

    @Override
    public void render() {
        // Очищаем экран (цвет темно-зеленый, как трава)
        ScreenUtils.clear(0.1f, 0.4f, 0.1f, 1); 
        
        // Обязательно обновляем камеру и соединяем её с отрисовкой
        camera.update();
        batch.setProjectionMatrix(camera.combined);

        float dt = Gdx.graphics.getDeltaTime();

        // --- 1. ЛОГИКА ИГРЫ ---

        // Таймер: каждые 0.8 секунды пытаемся показать монстра
        spawnTimer += dt;
        if (spawnTimer > 0.8f) { 
            spawnRandomMonster();
            spawnTimer = 0;
        }
        
        // Обновляем таймеры жизни у всех монстров
        for (Monster m : monsters) {
            m.update();
        }
        
        // Проверяем клик мышкой
        if (Gdx.input.justTouched()) {
            handleInput();
        }

        // --- 2. ОТРИСОВКА ---
        batch.begin();
        
        // Рисуем всех монстров (метод draw сам разберется, рисовать монстра или только нору)
        for (Monster m : monsters) {
            m.draw(batch);
        }
        
        batch.end();
    }
    
    // Выбираем случайную нору и говорим монстру: "Вылезай!"
    private void spawnRandomMonster() {
        int randomIndex = MathUtils.random(0, monsters.size() - 1);
        Monster m = monsters.get(randomIndex);
        
        // Если в этой норе монстр уже сидит, ничего не делаем
        if (!m.isVisible()) {
            m.show();
        }
    }

    // Самая сложная часть: Обработка клика
    private void handleInput() {
        // 1. Получаем координаты мыши на экране монитора
        int mouseX = Gdx.input.getX();
        int mouseY = Gdx.input.getY();
        
        // 2. Переводим их в координаты игрового мира с помощью камеры
        Vector3 touchPos = new Vector3(mouseX, mouseY, 0);
        camera.unproject(touchPos); 
        
        // 3. Проверяем каждого монстра
        for (Monster m : monsters) {
            // Если монстр виден И мы попали в него координатами из touchPos
            if (m.isVisible() && m.isHit(touchPos.x, touchPos.y)) {
                m.hide(); // Прячем монстра
                score++;  // Увеличиваем счет
                System.out.println("Попал! Твой счет: " + score);
            }
        }
    }

    @Override
    public void dispose() {
        batch.dispose();
        imgMonster.dispose();
        imgHole.dispose();
    }
}
```

---

## Запуск и проверка

1.  Запусти игру. Ты должен увидеть 9 нор.
2.  Из нор должны появляться монстры и исчезать сами через секунду.
3.  Кликай по монстрам! Если попал, в консоли (внизу в IntelliJ IDEA) должна появиться надпись: `"Попал! Твой счет: 1"`.

## Задания для самостоятельной работы

Попробуй улучшить игру:

1.  **Бомба:** Добавь картинку `bomb.png`. Сделай так, чтобы иногда вместо монстра появлялась бомба. Если кликнуть по ней — счет уменьшается на 5 очков!
2.  **Ускорение:** Сделай так, чтобы при наборе каждых 10 очков переменная `spawnTimer` становилась меньше (монстры начинают прыгать быстрее).
3.  **Game Over:** Если игрок пропустил 5 монстров (они исчезли сами, а не от клика), игра заканчивается или счет обнуляется. *Подсказка: нужно добавить проверку в метод `update` класса `Monster`.*
