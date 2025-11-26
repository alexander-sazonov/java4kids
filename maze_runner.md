---
layout: default
title: Проект: «Maze Runner»
parent: Уроки Java для детей
nav_order: 2
---

# 🏃‍♂️ Tutorial: Создание игры "Maze Runner"

**Уровень:** Средний
**Цель:** Создать героя, который бегает по лабиринту, собирает ключи и не может проходить сквозь стены.
**Новые концепции:**
*   Генерация уровня из массива строк (текстовая карта).
*   Продвинутая коллизия (скольжение вдоль стен).
*   Взаимодействие с предметами.

---

## 📂 Шаг 1: Подготовка (Assets)

Нам понадобятся квадратные картинки одинакового размера (например, **32x32** или **64x64** пикселя).
1.  `wall.png` — кирпичная стена.
2.  `hero.png` — наш персонаж.
3.  `coin.png` — монетка или ключ.
4.  `floor.png` — (опционально) пол, или просто зальем фон цветом.

> **Совет:** Скачайте пак "Top Down Shooter" или "Sokoban" с сайта kenney.nl, там есть всё необходимое.

---

## 🧱 Шаг 2: Класс "Стена" (Wall)

Начнем с простого. Стена — это просто картинка, которая стоит на месте и имеет "твердое тело" (hitbox).

```java
import com.badlogic.gdx.graphics.Texture;
import com.badlogic.gdx.graphics.g2d.SpriteBatch;
import com.badlogic.gdx.math.Rectangle;

class Wall {
    Rectangle hitbox;
    Texture image;

    public Wall(float x, float y, float size, Texture texture) {
        this.image = texture;
        this.hitbox = new Rectangle(x, y, size, size);
    }

    public void draw(SpriteBatch batch) {
        batch.draw(image, hitbox.x, hitbox.y, hitbox.width, hitbox.height);
    }
}
```

---

## 🦸‍♂️ Шаг 3: Класс "Герой" с Физикой

Это самый сложный и интересный класс. Герой должен проверять кнопки управления и **проверять, не врезался ли он в стену**.

Чтобы герой не застревал в стенах, мы применим хитрость "Двойная проверка":
1.  Двигаем по оси X -> Если врезались, отменяем движение по X.
2.  Двигаем по оси Y -> Если врезались, отменяем движение по Y.
Это позволит герою "скользить" вдоль стены.

```java
import com.badlogic.gdx.Gdx;
import com.badlogic.gdx.Input;
import com.badlogic.gdx.graphics.Texture;
import com.badlogic.gdx.graphics.g2d.SpriteBatch;
import com.badlogic.gdx.math.Rectangle;
import com.badlogic.gdx.utils.Array;

class Hero {
    Rectangle hitbox;
    Texture image;
    float speed = 200f; // Скорость бега

    public Hero(float x, float y, Texture texture) {
        this.image = texture;
        // Делаем хитбокс чуть меньше картинки (30x30), чтобы проще пролезать в проходы
        this.hitbox = new Rectangle(x, y, 30, 30);
    }

    // Метод update принимает список стен, чтобы знать, обо что биться
    public void update(float dt, Array<Wall> walls) {
        float oldX = hitbox.x;
        float oldY = hitbox.y;

        // 1. Движение по X (Влево-Вправо)
        if (Gdx.input.isKeyPressed(Input.Keys.LEFT)) hitbox.x -= speed * dt;
        if (Gdx.input.isKeyPressed(Input.Keys.RIGHT)) hitbox.x += speed * dt;

        // Проверка столкновений по X
        for (Wall wall : walls) {
            if (hitbox.overlaps(wall.hitbox)) {
                hitbox.x = oldX; // Отменяем движение, если врезались
                break; // Дальше проверять нет смысла
            }
        }

        // 2. Движение по Y (Вверх-Вниз)
        // Запоминаем X, так как он уже может быть новым (если не врезались)
        oldX = hitbox.x;

        if (Gdx.input.isKeyPressed(Input.Keys.UP)) hitbox.y += speed * dt;
        if (Gdx.input.isKeyPressed(Input.Keys.DOWN)) hitbox.y -= speed * dt;

        // Проверка столкновений по Y
        for (Wall wall : walls) {
            if (hitbox.overlaps(wall.hitbox)) {
                hitbox.y = oldY; // Отменяем движение по Y
                break;
            }
        }
    }

    public void draw(SpriteBatch batch) {
        batch.draw(image, hitbox.x, hitbox.y, hitbox.width, hitbox.height);
    }
}
```

---

## 🗺️ Шаг 4: Построение уровня (Главный класс)

Вместо того чтобы писать `new Wall(0,0)`, `new Wall(32,0)` сто раз, мы нарисуем карту текстом!
*   `#` — Стена
*   `.` — Пустота (пол)
*   `P` — Игрок (Player)
*   `*` — Монетка

```java
public class MazeGame extends ApplicationAdapter {
    SpriteBatch batch;
    Texture wallImg, heroImg, coinImg;

    Array<Wall> walls;
    Array<Rectangle> coins; // Монетки будут просто прямоугольниками
    Hero player;

    OrthographicCamera camera;

    // СХЕМА УРОВНЯ (1 строка = 1 ряд блоков)
    // 20 блоков шириной, 15 высотой. При блоке 32px это 640x480
    String[] map = {
        "####################",
        "#P.................#",
        "#.###.####.###.###.#",
        "#...#......#.....#.#",
        "###.######.#####.#.#",
        "#*..#....#.......#.#",
        "#.###.##.#######.#.#",
        "#......#...*.....#.#",
        "#.####.#########.#.#",
        "#....#...........#.#",
        "####################"
    };

    @Override
    public void create() {
        batch = new SpriteBatch();
        wallImg = new Texture("wall.png");
        heroImg = new Texture("hero.png");
        coinImg = new Texture("coin.png");

        walls = new Array<>();
        coins = new Array<>();

        // Размер одного блока (зависит от ваших картинок)
        float tileSize = 32;

        // --- ГЕНЕРАТОР УРОВНЯ ---
        // Читаем массив строк снизу вверх или сверху вниз
        // В LibGDX Y=0 внизу, поэтому проще читать массив с конца,
        // чтобы визуально карта совпадала с кодом.

        int rowCount = map.length;
        for (int y = 0; y < rowCount; y++) {
            // Берем строку (но переворачиваем порядок, чтобы map[0] был верхом)
            String line = map[rowCount - 1 - y];

            for (int x = 0; x < line.length(); x++) {
                char cell = line.charAt(x);

                if (cell == '#') {
                    walls.add(new Wall(x * tileSize, y * tileSize, tileSize, wallImg));
                }
                else if (cell == 'P') {
                    player = new Hero(x * tileSize, y * tileSize, heroImg);
                }
                else if (cell == '*') {
                    // Создаем монетку
                    Rectangle coinRect = new Rectangle(x * tileSize, y * tileSize, tileSize, tileSize);
                    coins.add(coinRect);
                }
            }
        }

        camera = new OrthographicCamera();
        camera.setToOrtho(false, 640, 480); // Размер окна под размер карты
    }

    @Override
    public void render() {
        ScreenUtils.clear(0, 0, 0, 1);

        // Обновляем логику героя
        player.update(Gdx.graphics.getDeltaTime(), walls);

        // Проверка сбора монет
        // Используем итератор для удаления собранных монет
        Iterator<Rectangle> iter = coins.iterator();
        while(iter.hasNext()) {
            Rectangle coin = iter.next();
            if (player.hitbox.overlaps(coin)) {
                iter.remove(); // Монетка собрана!
                System.out.println("Монета собрана!");
            }
        }

        camera.update();
        batch.setProjectionMatrix(camera.combined);

        batch.begin();

        // Рисуем стены
        for (Wall wall : walls) wall.draw(batch);

        // Рисуем монеты
        for (Rectangle coin : coins) {
            batch.draw(coinImg, coin.x, coin.y);
        }

        // Рисуем игрока
        player.draw(batch);

        batch.end();
    }

    @Override
    public void dispose() {
        batch.dispose();
        wallImg.dispose();
        heroImg.dispose();
        coinImg.dispose();
    }
}
```

---

## 💡 Как это работает?

1.  **Двойной цикл `for`:** Мы проходим по каждой букве в нашем текстовом массиве `map`.
2.  **`x * tileSize`:** Если мы нашли символ `#` на 5-й позиции в строке, а размер блока 32px, то координата стены будет `5 * 32 = 160`. Так мы превращаем буквы в пиксели.
3.  **Логика героя:** Герой получает список `walls` каждый кадр. Он пробует сделать шаг. Если шаг приводит к наложению прямоугольников (`overlaps`), шаг отменяется.

---

## 🔥 Задания для самостоятельной работы

Теперь превратите этот прототип в настоящую игру!

### Уровень 1: Выход (Easy)
Добавь в карту символ `E` (Exit/Выход).
*   Нарисуй для него картинку двери.
*   В коде создай `Rectangle exitRect`.
*   Условие победы: Если игрок касается выхода **И** количество монет (`coins.size`) равно 0 — выведи в консоль "YOU WIN!" (или закрой игру `Gdx.app.exit()`).

### Уровень 2: Камера слежения (Medium)
Сделай карту огромной (например, 50x50 символов). Она не поместится на экране.
*   Измени настройки камеры: `camera.position.set(player.hitbox.x, player.hitbox.y, 0);` внутри метода `render`.
*   Теперь камера будет всегда следовать за игроком, как в RPG.

### Уровень 3: Враги (Hard)
Добавь символ `Z` (Zombie).
*   Создай класс `Enemy` (похож на `Hero`, но двигается сам).
*   Простой ИИ: Зомби просто ходит влево-вправо. Если врезается в стену — меняет направление.
*   Если `hero.hitbox.overlaps(zombie.hitbox)` — игра начинается заново (`create()`).

### Уровень 4: Анимация (Pro)
Сделай так, чтобы герой поворачивался в сторону движения.
*   В классе `Hero` добавь поле `boolean flipX`.
*   При нажатии ВЛЕВО `flipX = true`, при нажатии ВПРАВО `flipX = false`.
*   В методе `draw` используй метод отрисовки с переворотом:
    ```java
    batch.draw(image, x, y, width, height, 0, 0, 32, 32, flipX, false);
    ```

Удачи в лабиринте! 🗝️🚪
