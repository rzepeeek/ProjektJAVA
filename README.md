# 🚀 CWL SIMULATOR

## 📌 Opis projektu
Projekt semestralny z przedmiotu **Programowanie Obiektowe II**  
Autorzy: **Adrian Czarnota 21442, Dawid Rzepka 21299**  
Data rozpoczęcia: *11.03.2026*  
Repozytorium zawiera kod oraz dokumentację projektu  

## 📂 Struktura projektu
```text
src/
 |── ⚙️ engine/
 |   |── 🎥 Camera
 |   |── 🔄 GameLoop
 |   |── 🎮 Input
 |   |── 🧱 Mesh
 |   |── 🖥️ Renderer
 |   |── 🎨 Shader
 |   |── 🪟 Window
 |   |── 🧵 Texture
 |── 🚗 entities/
 |   |── 🚘 Car
 |   |── 🎫 Ticket
 |── 🎮 game/
 |   |── 🕹️ Game
 |   |── 🧍 Player
 |── ⌨️ input/
 |── 🖼️ ui/
 |   |── 📊 HUD
 |── 🧮 utils/
 |   |── 🔢 Matrix4f
 |   |── ➡️ Vector3f
 |── 🚀 Main.java
```

## ⚙️ Engine
### 🎥 Camera
```java
package engine;

// ZMIENIONO IMPORT: Teraz używamy naszej własnej klasy z pakietu utils!
import utils.Vector3f;

public class Camera {
    // Wektor przechowujący pozycję kamery (X, Y, Z)
    private Vector3f position;
    // Wektor przechowujący obrót kamery (Pitch - góra/dół, Yaw - lewo/prawo, Roll - przechył)
    private Vector3f rotation;

    public Camera() {
        // Na start gry ustawiamy gracza w punkcie 0, 0, 0
        position = new Vector3f(0.0f, 0.0f, 0.0f);
        rotation = new Vector3f(0.0f, 0.0f, 0.0f);
    }

    // Metoda do poruszania graczem (np. pod klawisze W, A, S, D)
    public void movePosition(float offsetX, float offsetY, float offsetZ) {
        if (offsetZ != 0) {
            position.x += (float) Math.sin(Math.toRadians(rotation.y)) * -1.0f * offsetZ;
            position.z += (float) Math.cos(Math.toRadians(rotation.y)) * offsetZ;
        }
        if (offsetX != 0) {
            position.x += (float) Math.sin(Math.toRadians(rotation.y - 90)) * -1.0f * offsetX;
            position.z += (float) Math.cos(Math.toRadians(rotation.y - 90)) * offsetX;
        }
        position.y += offsetY;
    }

    // Metoda do obracania głową gracza (pod ruch myszką)
    public void moveRotation(float offsetX, float offsetY, float offsetZ) {
        rotation.x += offsetX;
        rotation.y += offsetY;
        rotation.z += offsetZ;

        // Zabezpieczenie, żeby gracz nie "złamał karku" patrząc za bardzo w górę lub w dół
        if (rotation.x > 90) rotation.x = 90;
        else if (rotation.x < -90) rotation.x = -90;
    }

    // Gettery do pobierania aktualnego stanu kamery
    public Vector3f getPosition() {
        return position;
    }

    public Vector3f getRotation() {
        return rotation;
    }
}
```

### 🔄 GameLoop
```java
package engine;

import static org.lwjgl.glfw.GLFW.*;
import static org.lwjgl.opengl.GL11.*;

public class GameLoop {
    private Window window;
    private boolean isRunning;
    private game.Game game;

    public GameLoop() {
        // Tworzymy okno gry o rozdzielczości 1280x720
        window = new Window("CWL Simulator", 1280, 720);
    }

    public void start() {
        window.init();
        isRunning = true;

        // Inicjalizacja logiki gry
        game = new game.Game();

        loop();
        cleanup();
    }

    private void loop() {
        // Zmienne do śledzenia czasu (Delta Time dla płynności 60 FPS)
        long lastTime = System.nanoTime();
        double targetFPS = 60.0;
        double nsPerFrame = 1000000000.0 / targetFPS;
        double delta = 0;

        while (isRunning && !window.windowShouldClose()) {
            long now = System.nanoTime();
            delta += (now - lastTime) / nsPerFrame;
            lastTime = now;

            // Aktualizacja logiki gry (stały krok czasowy)
            while (delta >= 1) {
                update();
                delta--;
            }

            // Renderowanie (rysowanie na ekranie)
            render();

            // Zamiana buforów i obsługa zdarzeń okna
            window.update();
        }
    }

    private void update() {
        // Zamknięcie gry klawiszem ESC
        if (Input.isKeyDown(GLFW_KEY_ESCAPE)) {
            glfwSetWindowShouldClose(window.getWindowHandle(), true);
        }

        // Aktualizacja pozycji gracza, kamery i sprawdzanie biletów
        game.update();

        // AKTUALIZACJA TYTUŁU OKNA - POBIERANIE DANYCH Z KLASY GAME
        // To jest Twój "awaryjny HUD", dopóki nie wgramy czcionek do niebieskiego pola
        String status = "CWL Simulator | Punkty: " + game.getScore() +
                " | Mandaty: " + game.getFinesIssued() +
                " | " + game.getUiMessage();

        glfwSetWindowTitle(window.getWindowHandle(), status);
    }

    private void render() {
        // Czyszczenie ekranu przed narysowaniem nowej klatki (Kolor tła i Głębia)
        glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT);

        // Wywołanie rysowania całego świata gry (Auta, Lidl, Podłoga, HUD)
        if (game != null) {
            game.render();
        }
    }

    private void cleanup() {
        window.cleanup();
    }
}
```

### 🎮 Input
```java
package engine;

import static org.lwjgl.glfw.GLFW.*;

public class Input {
    // Tablice przechowujące stan klawiszy i przycisków myszy (wciśnięty / puszczony)
    private static boolean[] keys = new boolean[GLFW_KEY_LAST];
    private static boolean[] buttons = new boolean[GLFW_MOUSE_BUTTON_LAST];

    // Zmienne do śledzenia pozycji myszy
    private static double mouseX, mouseY;

    // Metoda inicjalizująca - podpinamy się pod nasze okno
    public static void init(long windowHandle) {

        // Nasłuchiwacz klawiatury
        glfwSetKeyCallback(windowHandle, (window, key, scancode, action, mods) -> {
            if (key >= 0 && key < GLFW_KEY_LAST) {
                // Jeśli akcja to nie "puszczenie klawisza", oznaczamy go jako wciśnięty (true)
                keys[key] = (action != GLFW_RELEASE);
            }
        });

        // Nasłuchiwacz ruchu myszy
        glfwSetCursorPosCallback(windowHandle, (window, xpos, ypos) -> {
            mouseX = xpos;
            mouseY = ypos;
        });

        // Nasłuchiwacz kliknięć myszy
        glfwSetMouseButtonCallback(windowHandle, (window, button, action, mods) -> {
            if (button >= 0 && button < GLFW_MOUSE_BUTTON_LAST) {
                buttons[button] = (action != GLFW_RELEASE);
            }
        });
    }

    // Funkcja do sprawdzania w innych miejscach kodu, czy dany klawisz jest wciśnięty
    public static boolean isKeyDown(int key) {
        return keys[key];
    }

    // Funkcja do sprawdzania, czy przycisk myszy jest wciśnięty
    public static boolean isButtonDown(int button) {
        return buttons[button];
    }

    // Pobieranie pozycji X myszy
    public static double getMouseX() {
        return mouseX;
    }

    // Pobieranie pozycji Y myszy
    public static double getMouseY() {
        return mouseY;
    }
}
```

### 🧱 Mesh
```java
package engine;

import static org.lwjgl.opengl.GL30.*;

public class Mesh {
    private int vaoId;
    private int vertexCount;

    public Mesh(float[] vertices) {
        // Obliczamy ile punktów ma kształt (każdy punkt to 3 liczby: X, Y, Z)
        vertexCount = vertices.length / 3;

        // Generujemy VAO i VBO (bufory karty graficznej)
        vaoId = glGenVertexArrays();
        glBindVertexArray(vaoId);

        int vboId = glGenBuffers();
        glBindBuffer(GL_ARRAY_BUFFER, vboId);
        glBufferData(GL_ARRAY_BUFFER, vertices, GL_STATIC_DRAW);

        glVertexAttribPointer(0, 3, GL_FLOAT, false, 0, 0);

        // Odpinamy bufory
        glBindBuffer(GL_ARRAY_BUFFER, 0);
        glBindVertexArray(0);
    }

    public int getVaoId() {
        return vaoId;
    }

    public int getVertexCount() {
        return vertexCount;
    }
}
```

### 🖥️ Renderer
```java
package engine;

import static org.lwjgl.opengl.GL30.*;
import utils.Matrix4f;
import utils.Vector3f;
import entities.Car;
import java.util.List;

public class Renderer {
    private Shader shader;
    private int vaoId;

    public void init() {
        String vertexShader = "#version 330 core\n" +
                "layout (location = 0) in vec3 position;\n" +
                "uniform mat4 projection;\n" +
                "uniform mat4 view;\n" +
                "uniform mat4 model;\n" +
                "out vec2 texCoord;\n" +
                "void main() {\n" +
                "  gl_Position = projection * view * model * vec4(position, 1.0);\n" +
                "  texCoord = position.xy * 0.5 + 0.5;\n" +
                "}";

        String fragmentShader = "#version 330 core\n" +
                "uniform vec4 color;\n" +
                "uniform sampler2D tex;\n" +
                "uniform int useTexture;\n" +
                "in vec2 texCoord;\n" +
                "out vec4 fragColor;\n" +
                "void main() {\n" +
                "  if(useTexture == 1) {\n" +
                "    fragColor = texture(tex, vec2(texCoord.x, 1.0 - texCoord.y)) * color;\n" +
                "  } else {\n" +
                "    fragColor = color;\n" +
                "  }\n" +
                "}";

        shader = new Shader(vertexShader, fragmentShader);

        float[] vertices = {
                -0.5f, 0.5f, 0.0f,  -0.5f, -0.5f, 0.0f,   0.5f, -0.5f, 0.0f,
                0.5f, -0.5f, 0.0f,   0.5f, 0.5f, 0.0f,  -0.5f, 0.5f, 0.0f
        };
        vaoId = glGenVertexArrays();
        glBindVertexArray(vaoId);
        int vboId = glGenBuffers();
        glBindBuffer(GL_ARRAY_BUFFER, vboId);
        glBufferData(GL_ARRAY_BUFFER, vertices, GL_STATIC_DRAW);
        glVertexAttribPointer(0, 3, GL_FLOAT, false, 0, 0);
        glBindVertexArray(0);
    }

    public void render(Camera camera, Mesh floorMesh, Mesh lineMesh, List<Car> cars) {
        shader.bind();
        // KLUCZOWE: Wyłączamy tekstury dla świata 3D
        shader.setUniform("useTexture", 0);

        Matrix4f projection = new Matrix4f();
        projection.setPerspective(70.0f, 1280.0f / 720.0f, 0.1f, 1000.0f);
        shader.setUniform("projection", projection);

        Matrix4f view = new Matrix4f();
        view.rotateX(camera.getRotation().x);
        view.rotateY(camera.getRotation().y);
        view.translate(-camera.getPosition().x, -camera.getPosition().y, -camera.getPosition().z);
        shader.setUniform("view", view);

        // Podłoga
        shader.setUniform("color", 0.2f, 0.6f, 0.2f, 1.0f);
        Matrix4f floorModel = new Matrix4f();
        floorModel.translate(0.0f, -1.5f, -5.0f);
        floorModel.scale(40.0f, 1.0f, 40.0f);
        shader.setUniform("model", floorModel);
        glBindVertexArray(floorMesh.getVaoId());
        glEnableVertexAttribArray(0);
        glDrawArrays(GL_TRIANGLES, 0, floorMesh.getVertexCount());

        // Linie
        shader.setUniform("color", 0.9f, 0.9f, 0.9f, 1.0f);
        glBindVertexArray(lineMesh.getVaoId());
        for (int i = 0; i < 11; i++) {
            Matrix4f lineModel = new Matrix4f();
            lineModel.translate(-15.0f + (i * 3.0f), -1.49f, -8.0f);
            shader.setUniform("model", lineModel);
            glDrawArrays(GL_TRIANGLES, 0, lineMesh.getVertexCount());
        }

        // Auta
        for (Car car : cars) {
            shader.setUniform("color", car.getColor().x, car.getColor().y, car.getColor().z, 1.0f);
            Matrix4f carModel = new Matrix4f();
            carModel.translate(car.getPosition().x, car.getPosition().y, car.getPosition().z);
            shader.setUniform("model", carModel);
            glBindVertexArray(car.getMesh().getVaoId());
            glEnableVertexAttribArray(0);
            glDrawArrays(GL_TRIANGLES, 0, car.getMesh().getVertexCount());
        }

        glDisableVertexAttribArray(0);
        glBindVertexArray(0);
        shader.unbind();
    }

    public void renderUIQuad(float x, float y, float width, float height, Vector3f color, int useTexture) {
        shader.bind();
        Matrix4f identity = new Matrix4f();
        shader.setUniform("projection", identity);
        shader.setUniform("view", identity);

        shader.setUniform("color", color.x, color.y, color.z, 0.7f);
        shader.setUniform("useTexture", useTexture);

        Matrix4f model = new Matrix4f();
        model.translate(x, y, 0);
        model.scale(width, height, 1);
        shader.setUniform("model", model);

        glBindVertexArray(vaoId);
        glEnableVertexAttribArray(0);
        glDrawArrays(GL_TRIANGLES, 0, 6);
        glDisableVertexAttribArray(0);
        glBindVertexArray(0);

        // NAPRAWIONE: Dodano drugi argument (0), aby odpiąć teksturę
        glBindTexture(GL_TEXTURE_2D, 0);
        shader.unbind();
    }

    public void renderMesh(Camera camera, Mesh mesh, Matrix4f modelMatrix, Vector3f color) {
        shader.bind();
        shader.setUniform("useTexture", 0); // Wyłącz tekstury dla meshów (np. Lidl)

        Matrix4f projection = new Matrix4f();
        projection.setPerspective(70.0f, 1280.0f / 720.0f, 0.1f, 1000.0f);
        shader.setUniform("projection", projection);

        Matrix4f view = new Matrix4f();
        view.rotateX(camera.getRotation().x);
        view.rotateY(camera.getRotation().y);
        view.translate(-camera.getPosition().x, -camera.getPosition().y, -camera.getPosition().z);
        shader.setUniform("view", view);

        shader.setUniform("model", modelMatrix);
        shader.setUniform("color", color.x, color.y, color.z, 1.0f);

        glBindVertexArray(mesh.getVaoId());
        glEnableVertexAttribArray(0);
        glDrawArrays(GL_TRIANGLES, 0, mesh.getVertexCount());
        glDisableVertexAttribArray(0);
        glBindVertexArray(0);
    }
}
```

### 🎨 Shader
```java
package engine;

import static org.lwjgl.opengl.GL20.*;

public class Shader {
    private int programId;

    public Shader(String vertexCode, String fragmentCode) {
        // Tworzymy główny program cieniujący
        programId = glCreateProgram();

        // Tworzymy i kompilujemy Vertex Shader (odpowiada za pozycję na ekranie)
        int vertexShaderId = glCreateShader(GL_VERTEX_SHADER);
        glShaderSource(vertexShaderId, vertexCode);
        glCompileShader(vertexShaderId);
        glAttachShader(programId, vertexShaderId);

        // Tworzymy i kompilujemy Fragment Shader (odpowiada za kolorowanie - np. farba na autach)
        int fragmentShaderId = glCreateShader(GL_FRAGMENT_SHADER);
        glShaderSource(fragmentShaderId, fragmentCode);
        glCompileShader(fragmentShaderId);
        glAttachShader(programId, fragmentShaderId);

        // Łączymy to w jedną całość
        glLinkProgram(programId);
    }

    // Włączamy shader przed rysowaniem
    public void bind() {
        glUseProgram(programId);
    }

    // Wyłączamy po narysowaniu
    public void unbind() {
        glUseProgram(0);
    }

    // ... poprzedni kod Shadera ...

    // DODAJ TĘ METODĘ NA SAMYM DOLE (przed zamykającym nawiasem `}` klasy):
    public void setUniform(String name, utils.Matrix4f matrix) {
        int location = org.lwjgl.opengl.GL20.glGetUniformLocation(programId, name);
        org.lwjgl.opengl.GL20.glUniformMatrix4fv(location, false, matrix.m);
    }

    // Metoda do ustawiania koloru (R - czerwony, G - zielony, B - niebieski, A - przezroczystość)
    public void setUniform(String name, float r, float g, float b, float a) {
        int location = org.lwjgl.opengl.GL20.glGetUniformLocation(programId, name);
        org.lwjgl.opengl.GL20.glUniform4f(location, r, g, b, a);
    }

    // DODAJ TO TUTAJ - Metoda do przesyłania liczb całkowitych (0 lub 1)
    public void setUniform(String name, int value) {
        int location = org.lwjgl.opengl.GL20.glGetUniformLocation(programId, name);
        if (location != -1) {
            org.lwjgl.opengl.GL20.glUniform1i(location, value);
        }
    }
}
```

### 🪟 Window
```java
package engine;

import org.lwjgl.glfw.GLFWErrorCallback;
import org.lwjgl.opengl.GL;
import static org.lwjgl.glfw.GLFW.*;
import static org.lwjgl.opengl.GL11.*;
import static org.lwjgl.system.MemoryUtil.NULL;

public class Window {
    private String title;
    private int width, height;
    private long windowHandle;

    public Window(String title, int width, int height) {
        this.title = title;
        this.width = width;
        this.height = height;
    }

    // Inicjalizacja okna GLFW i OpenGL
    public void init() {
        // Konfiguracja obsługi błędów
        GLFWErrorCallback.createPrint(System.err).set();

        // Inicjalizacja biblioteki GLFW
        if (!glfwInit()) {
            throw new IllegalStateException("Nie udało się zainicjować GLFW!");
        }

        // Konfiguracja parametrów okna
        glfwDefaultWindowHints();
        glfwWindowHint(GLFW_VISIBLE, GLFW_FALSE); // Okno ukryte po utworzeniu
        glfwWindowHint(GLFW_RESIZABLE, GLFW_TRUE); // Możliwość zmiany rozmiaru

        // Tworzenie okna
        windowHandle = glfwCreateWindow(width, height, title, NULL, NULL);
        if (windowHandle == NULL) {
            throw new RuntimeException("Nie udało się utworzyć okna GLFW!");
        }

        // Ustawienie kontekstu OpenGL na nasze okno
        glfwMakeContextCurrent(windowHandle);

        // Włączenie V-Sync (synchronizacja pionowa)
        glfwSwapInterval(1);

        // Pokazanie okna
        glfwShowWindow(windowHandle);

        // Krytyczna linijka w LWJGL! Wiąże kontekst GLFW z OpenGL
        GL.createCapabilities();

        // Ustawienie koloru tła (czyszczenia ekranu) - np. jasnoniebieskie niebo
        glClearColor(0.5f, 0.7f, 0.9f, 1.0f);
        glEnable(GL_DEPTH_TEST);

        glEnable(GL_BLEND);
        glBlendFunc(GL_SRC_ALPHA, GL_ONE_MINUS_SRC_ALPHA);
        // Inicjalizacja systemu sterowania na samym końcu, gdy okno już istnieje!
        Input.init(windowHandle);
        // Blokujemy i ukrywamy kursor myszy (Tryb FPS)
        org.lwjgl.glfw.GLFW.glfwSetInputMode(windowHandle, org.lwjgl.glfw.GLFW.GLFW_CURSOR, org.lwjgl.glfw.GLFW.GLFW_CURSOR_DISABLED);
    }

    // Metoda wywoływana co klatkę, aby zaktualizować ekran
    public void update() {
        glfwSwapBuffers(windowHandle); // Zamiana buforów (podwójne buforowanie)
        glfwPollEvents();              // Sprawdzanie zdarzeń (np. klawiatury/myszy)
    }

    // Sprawdza, czy użytkownik kliknął "X" aby zamknąć okno
    public boolean windowShouldClose() {
        return glfwWindowShouldClose(windowHandle);
    }

    // Sprzątanie pamięci po zamknięciu gry
    public void cleanup() {
        glfwDestroyWindow(windowHandle);
        glfwTerminate();
        glfwSetErrorCallback(null).free();
    }

    // Zwraca ID naszego okna (potrzebne m.in. do Inputu)
    public long getWindowHandle() {
        return windowHandle;
    }
}
```

### 🧵 Texture
```java
package engine;

import static org.lwjgl.opengl.GL11.*;
import java.nio.ByteBuffer;
import java.awt.image.BufferedImage;
import java.awt.Graphics2D;
import java.awt.Color;
import java.awt.Font;
import org.lwjgl.BufferUtils;

public class Texture {
    private int id;

    public Texture(String text) {
        // Tworzymy obrazek w pamięci RAM (Java AWT)
        BufferedImage img = new BufferedImage(800, 64, BufferedImage.TYPE_INT_ARGB);
        Graphics2D g = img.createGraphics();
        g.setRenderingHint(java.awt.RenderingHints.KEY_ANTIALIASING, java.awt.RenderingHints.VALUE_ANTIALIAS_ON); // Wygładzanie
        g.setFont(new Font("Arial", Font.BOLD, 30));
        g.setColor(Color.WHITE);
        g.drawString(text, 10, 45);
        g.dispose();

        // Zamieniamy obrazek na dane zrozumiałe dla karty graficznej
        int[] pixels = new int[img.getWidth() * img.getHeight()];
        img.getRGB(0, 0, img.getWidth(), img.getHeight(), pixels, 0, img.getWidth());
        ByteBuffer buffer = BufferUtils.createByteBuffer(img.getWidth() * img.getHeight() * 4);

        for(int y = 0; y < img.getHeight(); y++){
            for(int x = 0; x < img.getWidth(); x++){
                int pixel = pixels[y * img.getWidth() + x];
                buffer.put((byte) ((pixel >> 16) & 0xFF));
                buffer.put((byte) ((pixel >> 8) & 0xFF));
                buffer.put((byte) (pixel & 0xFF));
                buffer.put((byte) ((pixel >> 24) & 0xFF));
            }
        }
        buffer.flip();

        id = glGenTextures();
        glBindTexture(GL_TEXTURE_2D, id);
        glTexImage2D(GL_TEXTURE_2D, 0, GL_RGBA, img.getWidth(), img.getHeight(), 0, GL_RGBA, GL_UNSIGNED_BYTE, buffer);
        glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_LINEAR);
        glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_LINEAR);
    }

    public void bind() { glBindTexture(GL_TEXTURE_2D, id); }
    public void delete() { glDeleteTextures(id); }
}
```

## 🚗 Entities
### 🚘 Car
```java
package entities;

import engine.Mesh;
import utils.Vector3f;

public class Car {
    private Mesh mesh;
    private Vector3f position;
    private Vector3f color;
    private Ticket ticket;
    private boolean hasFine = false; // <-- DODAJ TO TUTAJ

    public Car(Mesh mesh, Vector3f position, Vector3f color, Ticket ticket) {
        this.mesh = mesh;
        this.position = position;
        this.color = color;
        this.ticket = ticket;
    }

    // --- DODAJ TE DWIE METODY ---
    public boolean hasFine() {
        return hasFine;
    }

    public void setHasFine(boolean hasFine) {
        this.hasFine = hasFine;
    }
    // ----------------------------

    public Mesh getMesh() { return mesh; }
    public Vector3f getPosition() { return position; }
    public Vector3f getColor() { return color; }
    public Ticket getTicket() { return ticket; }
}
```

### 🎫 Ticket
```java
package entities;

public class Ticket {
    private long startTime;       // Czas zaparkowania (w milisekundach)
    private long maxParkingTime;  // Dozwolony czas parkowania (w milisekundach)

    public Ticket(long maxParkingTimeMillis) {
        // System.currentTimeMillis() pobiera aktualny czas z zegara komputera
        this.startTime = System.currentTimeMillis();
        this.maxParkingTime = maxParkingTimeMillis;
    }

    // Metoda sprawdzająca, czy bilet jest przeterminowany
    public boolean isExpired() {
        long currentTime = System.currentTimeMillis();
        return (currentTime - startTime) > maxParkingTime;
    }

    // Zwraca ile sekund zostało do końca biletu (przydatne do wyświetlania)
    public long getRemainingSeconds() {
        long elapsed = System.currentTimeMillis() - startTime;
        long remainingMillis = maxParkingTime - elapsed;

        if (remainingMillis < 0) return 0; // Jeśli czas minął, zwracamy 0
        return remainingMillis / 1000;     // Dzielimy przez 1000, żeby zamienić milisekundy na sekundy
    }
}
```

## 🎮 Game
### 🕹️ Game
```java
package game;

import engine.Renderer;
import engine.Mesh;
import engine.Input;
import entities.Car;
import entities.Ticket;
import utils.Vector3f;
import static org.lwjgl.glfw.GLFW.*;
import ui.HUD;
import utils.Matrix4f;

import java.util.ArrayList;
import java.util.List;

public class Game {
    private Player player;
    private Renderer renderer;
    private Mesh floorMesh;
    private Mesh carMesh;
    private List<Car> cars;
    private Mesh lineMesh;
    private HUD hud;

    // --- ZMIENNE PUNKTACJI I SYSTEMU ---
    private int score = 0;
    private int finesIssued = 0;
    private long lastInteractTime = 0;

    private String uiMessage = "Witaj w pracy, kontrolerze!";
    private long messageDisplayStartTime = 0;
    private final long MESSAGE_DURATION = 3000;

    // --- DODANE METODY GETTER (NAPRAWA BŁĘDU W GAMELOOP) ---
    public int getScore() {
        return score;
    }

    public int getFinesIssued() {
        return finesIssued;
    }

    public String getUiMessage() {
        if (System.currentTimeMillis() - messageDisplayStartTime > MESSAGE_DURATION) return "";
        return uiMessage;
    }
    // -------------------------------------------------------

    public Game() {
        player = new Player();
        renderer = new Renderer();
        renderer.init();
        hud = new HUD();

        // Model Podłogi
        float[] floorVertices = {
                -0.5f, 0.0f, -0.5f,  -0.5f, 0.0f,  0.5f,   0.5f, 0.0f,  0.5f,
                0.5f, 0.0f,  0.5f,   0.5f, 0.0f, -0.5f,  -0.5f, 0.0f, -0.5f
        };
        floorMesh = new Mesh(floorVertices);

        // Model Sześcianu (Auta/Budynki)
        float[] cubeVertices = {
                -0.5f, -0.5f,  0.5f,   0.5f, -0.5f,  0.5f,   0.5f,  0.5f,  0.5f,
                0.5f,  0.5f,  0.5f,  -0.5f,  0.5f,  0.5f,  -0.5f, -0.5f,  0.5f,
                -0.5f, -0.5f, -0.5f,  -0.5f,  0.5f, -0.5f,   0.5f,  0.5f, -0.5f,
                0.5f,  0.5f, -0.5f,   0.5f, -0.5f, -0.5f,  -0.5f, -0.5f, -0.5f,
                -0.5f,  0.5f,  0.5f,  -0.5f,  0.5f, -0.5f,  -0.5f, -0.5f, -0.5f,
                -0.5f, -0.5f, -0.5f,  -0.5f, -0.5f,  0.5f,  -0.5f,  0.5f,  0.5f,
                0.5f,  0.5f,  0.5f,   0.5f, -0.5f, -0.5f,   0.5f,  0.5f, -0.5f,
                0.5f, -0.5f, -0.5f,   0.5f,  0.5f,  0.5f,   0.5f, -0.5f,  0.5f,
                -0.5f,  0.5f, -0.5f,  -0.5f,  0.5f,  0.5f,   0.5f,  0.5f,  0.5f,
                0.5f,  0.5f,  0.5f,   0.5f,  0.5f, -0.5f,  -0.5f,  0.5f, -0.5f,
                -0.5f, -0.5f, -0.5f,   0.5f, -0.5f, -0.5f,   0.5f, -0.5f,  0.5f,
                0.5f, -0.5f,  0.5f,  -0.5f, -0.5f,  0.5f,  -0.5f, -0.5f, -0.5f
        };
        carMesh = new Mesh(cubeVertices);

        // Model Linii
        float[] lineVertices = {
                -0.05f, 0.01f, -2.0f,  -0.05f, 0.01f,  2.0f,   0.05f, 0.01f,  2.0f,
                0.05f, 0.01f,  2.0f,   0.05f, 0.01f, -2.0f,  -0.05f, 0.01f, -2.0f
        };
        lineMesh = new Mesh(lineVertices);

        cars = new ArrayList<>();
        cars.add(new Car(carMesh, new Vector3f(0.0f, -1.0f, -5.0f), new Vector3f(0.8f, 0.1f, 0.1f), new Ticket(10000)));
        cars.add(new Car(carMesh, new Vector3f(-3.0f, -1.0f, -5.0f), new Vector3f(0.1f, 0.1f, 0.8f), new Ticket(60000)));
        cars.add(new Car(carMesh, new Vector3f(3.0f, -1.0f, -5.0f), new Vector3f(0.8f, 0.8f, 0.1f), new Ticket(120000)));
    }

    public void update() {
        player.update();
        if (Input.isKeyDown(GLFW_KEY_E)) handleInteraction(false);
        if (Input.isKeyDown(GLFW_KEY_F)) handleInteraction(true);
    }

    private void handleInteraction(boolean isIssuingFine) {
        long currentTime = System.currentTimeMillis();
        if (currentTime - lastInteractTime > 500) {
            Car target = getClosestCar(2.5f);
            if (target != null) {
                if (isIssuingFine) {
                    issueFine(target); // Wywołujemy poprawione issueFine
                } else {
                    checkTicket(target);
                }
            }
            lastInteractTime = currentTime;
        }
    }

    private Car getClosestCar(float maxDistance) {
        Car closest = null;
        float minDistance = maxDistance;
        for (Car car : cars) {
            float dx = player.getCamera().getPosition().x - car.getPosition().x;
            float dz = player.getCamera().getPosition().z - car.getPosition().z;
            float distance = (float) Math.sqrt(dx * dx + dz * dz);
            if (distance < minDistance) {
                minDistance = distance;
                closest = car;
            }
        }
        return closest;
    }

    private void checkTicket(Car car) {
        if (car.getTicket().isExpired()) showUIMessage("BILET PRZETERMINOWANY! [F] - Mandat");
        else showUIMessage("Bilet ważny. Zostało: " + car.getTicket().getRemainingSeconds() + "s");
    }

    private void issueFine(Car car) {
        // Blokada: jeśli auto ma już mandat, kończymy robotę
        if (car.hasFine()) {
            showUIMessage("TEN SAMOCHÓD MA JUŻ MANDAT!");
            return;
        }

        if (car.getTicket().isExpired()) {
            score += 10;
            finesIssued++;
            car.setHasFine(true); // Oznaczamy auto jako "ukarane"
            showUIMessage("POPRAWNY MANDAT! +10 PKT");
        } else {
            score -= 15;
            showUIMessage("BŁĄD! MANDAT NIESŁUSZNY! -15 PKT");
        }
    }

    private void showUIMessage(String msg) {
        this.uiMessage = msg;
        this.messageDisplayStartTime = System.currentTimeMillis();
        System.out.println(msg);
    }

    public void render() {
        // 1. RYSOWANIE ŚWIATA (To co już masz)
        renderer.render(player.getCamera(), floorMesh, lineMesh, cars);

        // LIDL
        Matrix4f lidlWall = new Matrix4f();
        lidlWall.translate(0.0f, 2.5f, -20.0f);
        lidlWall.scale(30.0f, 8.0f, 1.0f);
        renderer.renderMesh(player.getCamera(), carMesh, lidlWall, new Vector3f(0.6f, 0.6f, 0.6f));

        // 2. OBLICZENIA DLA HUD-a (Naprawa błędu cannot find symbol)
        String ticketStatus = "BRAK"; // <--- TUTAJ JEST TA DEKLARACJA
        entities.Car closest = getClosestCar(2.5f);
        if (closest != null) {
            ticketStatus = closest.getTicket().getRemainingSeconds() + "s";
        }

        // Pobieramy komunikat (ten na środek ekranu)
        String currentMsg = getUiMessage();

        // 3. RYSOWANIE HUD (Zawsze na końcu)
        org.lwjgl.opengl.GL11.glDisable(org.lwjgl.opengl.GL11.GL_DEPTH_TEST);

        // Przekazujemy wszystkie 5 argumentów: renderer, score, finesIssued, ticketStatus, currentMsg
        hud.drawHUD(renderer, score, finesIssued, ticketStatus, currentMsg);

        org.lwjgl.opengl.GL11.glEnable(org.lwjgl.opengl.GL11.GL_DEPTH_TEST);
    }
}
```

### 🧍 Player
```java
package game;

import engine.Camera;
import engine.Input;
import static org.lwjgl.glfw.GLFW.*;

public class Player {
    private Camera camera;

    // Zmienne do płynnego sterowania
    private float speed = 0.1f;
    private float mouseSensitivity = 0.15f; // Czułość myszki

    // Zmienne do śledzenia ruchu myszy
    private double oldMouseX, oldMouseY;
    private boolean firstMouse = true; // Flaga chroniąca przed gwałtownym skokiem kamery na start

    public Player() {
        camera = new Camera();
    }

    public void update() {
        // --- 1. OBRACANIE KAMERY (MYSZKA) ---
        double newMouseX = Input.getMouseX();
        double newMouseY = Input.getMouseY();

        // Jeśli to pierwsza klatka gry, ustawiamy "starą" pozycję na obecną
        if (firstMouse) {
            oldMouseX = newMouseX;
            oldMouseY = newMouseY;
            firstMouse = false;
        }

        // Obliczamy różnicę w ruchu myszy (Delta)
        float dx = (float) (newMouseX - oldMouseX);
        float dy = (float) (newMouseY - oldMouseY);

        // Zapisujemy aktualną pozycję na następną klatkę
        oldMouseX = newMouseX;
        oldMouseY = newMouseY;

        // Przekazujemy ruch do kamery: dy (oś Y ekranu) obraca głowę w górę/dół (Pitch)
        // dx (oś X ekranu) obraca głowę w lewo/prawo (Yaw)
        camera.moveRotation(dy * mouseSensitivity, dx * mouseSensitivity, 0);


        // --- 2. CHODZENIE (KLAWIATURA) ---
        // Zauważ, że dzięki naszej matematyce w Camera.java, chodzenie "do przodu" (W)
        // teraz automatycznie podąża za tym, gdzie patrzysz!
        if (Input.isKeyDown(GLFW_KEY_W)) {
            camera.movePosition(0, 0, -speed);
        }
        if (Input.isKeyDown(GLFW_KEY_S)) {
            camera.movePosition(0, 0, speed);
        }
        if (Input.isKeyDown(GLFW_KEY_A)) {
            camera.movePosition(-speed, 0, 0);
        }
        if (Input.isKeyDown(GLFW_KEY_D)) {
            camera.movePosition(speed, 0, 0);
        }

        // Opcjonalnie: Bieg pod klawiszem SHIFT (zgodnie z Twoim pomysłem!)
        if (Input.isKeyDown(GLFW_KEY_LEFT_SHIFT)) {
            speed = 0.25f; // Szybszy krok
        } else {
            speed = 0.1f;  // Normalny krok
        }
    }

    public Camera getCamera() {
        return camera;
    }
}
```

## ⌨️ Input

## 🖼️ UI
### 📊 HUD
```java
package ui;

import engine.Renderer;
import engine.Texture;
import utils.Vector3f;

public class HUD {
    private Texture lastHudTexture;
    private String lastHudText = "";

    private Texture messageTexture;
    private String lastMessageText = "";

    public void drawHUD(Renderer renderer, int score, int fines, String ticketInfo, String uiMessage) {
        // --- 1. STAŁY PANEL (Górny Lewy Róg) ---
        // Rysujemy tło (0 - bez tekstury)
        renderer.renderUIQuad(-0.75f, 0.9f, 0.45f, 0.15f, new Vector3f(0.0f, 0.3f, 0.7f), 0);

        String hudText = "PKT: " + score + " | BILET: " + ticketInfo;

        // Odświeżamy obrazek napisu tylko gdy tekst się zmieni (żeby gra nie mulila)
        if (!hudText.equals(lastHudText)) {
            if (lastHudTexture != null) lastHudTexture.delete();
            lastHudTexture = new Texture(hudText);
            lastHudText = hudText;
        }

        // Naklejamy napis na prostokąt (1 - z teksturą)
        if (lastHudTexture != null) {
            lastHudTexture.bind();
            renderer.renderUIQuad(-0.72f, 0.88f, 0.4f, 0.06f, new Vector3f(1, 1, 1), 1);
            org.lwjgl.opengl.GL11.glBindTexture(org.lwjgl.opengl.GL11.GL_TEXTURE_2D, 0); // Odepnij zaraz po użyciu!
        }

        // --- 2. KOMUNIKAT NA ŚRODKU (Dynamiczny) ---
        // Rysujemy tylko jeśli istnieje komunikat!
        if (uiMessage != null && !uiMessage.trim().isEmpty()) {
            if (!uiMessage.equals(lastMessageText)) {
                if (messageTexture != null) messageTexture.delete();
                messageTexture = new Texture(uiMessage);
                lastMessageText = uiMessage;
            }
            if (messageTexture != null) {
                messageTexture.bind();
                // X: 0.0f, szerokość: DUŻA (1.2f), żeby wszedł cały napis
                renderer.renderUIQuad(0.0f, -0.2f, 1.2f, 0.12f, new Vector3f(1, 1, 1), 1);
                org.lwjgl.opengl.GL11.glBindTexture(org.lwjgl.opengl.GL11.GL_TEXTURE_2D, 0); // Odepnij!
            }
        }

        // --- 3. CELOWNIK ---
        renderer.renderUIQuad(0, 0, 0.01f, 0.01f, new Vector3f(1, 1, 1), 0);
    }
}
```

## 🧮 Utils
### 🔢 Matrix4f
```java
package utils;

public class Matrix4f {
    // 16 liczb, które karta graficzna rozumie jako przestrzeń 3D
    public float[] m = new float[16];

    public Matrix4f() {
        setIdentity();
    }

    // Resetowanie macierzy (ustawienie "na zero")
    public void setIdentity() {
        for (int i = 0; i < 16; i++) m[i] = 0;
        m[0] = 1; m[5] = 1; m[10] = 1; m[15] = 1;
    }

    // Matematyka 3D: Mnożenie macierzy
    public void multiply(Matrix4f other) {
        float[] result = new float[16];
        for (int col = 0; col < 4; col++) {
            for (int row = 0; row < 4; row++) {
                result[row + col * 4] =
                        this.m[row + 0 * 4] * other.m[0 + col * 4] +
                                this.m[row + 1 * 4] * other.m[1 + col * 4] +
                                this.m[row + 2 * 4] * other.m[2 + col * 4] +
                                this.m[row + 3 * 4] * other.m[3 + col * 4];
            }
        }
        this.m = result;
    }

    // Przesuwanie obiektów (np. naszego parkingu w dół)
    public void translate(float x, float y, float z) {
        Matrix4f trans = new Matrix4f();
        trans.m[12] = x; trans.m[13] = y; trans.m[14] = z;
        this.multiply(trans);
    }

    // Skalowanie (np. zrobienie ogromnego parkingu z małego kwadratu)
    public void scale(float x, float y, float z) {
        Matrix4f scaleMat = new Matrix4f();
        scaleMat.m[0] = x; scaleMat.m[5] = y; scaleMat.m[10] = z;
        this.multiply(scaleMat);
    }

    // Obracanie kamery w górę/dół (oś X)
    public void rotateX(float angle) {
        Matrix4f rot = new Matrix4f();
        float rad = (float) Math.toRadians(angle);
        float cos = (float) Math.cos(rad);
        float sin = (float) Math.sin(rad);
        rot.m[5] = cos; rot.m[6] = sin;
        rot.m[9] = -sin; rot.m[10] = cos;
        this.multiply(rot);
    }

    // Obracanie kamery w lewo/prawo (oś Y)
    public void rotateY(float angle) {
        Matrix4f rot = new Matrix4f();
        float rad = (float) Math.toRadians(angle);
        float cos = (float) Math.cos(rad);
        float sin = (float) Math.sin(rad);
        rot.m[0] = cos; rot.m[2] = -sin;
        rot.m[8] = sin; rot.m[10] = cos;
        this.multiply(rot);
    }

    // Generator Perspektywy (to dzięki temu obiekty w oddali są mniejsze!)
    // Generator Perspektywy (Naprawiony!)
    public void setPerspective(float fov, float aspect, float zNear, float zFar) {
        setIdentity();
        float tanHalfFOV = (float) Math.tan(Math.toRadians(fov / 2f));
        float zRange = zFar - zNear; // POPRAWKA: zFar minus zNear (wynik dodatni)

        m[0] = 1.0f / (tanHalfFOV * aspect);
        m[5] = 1.0f / tanHalfFOV;
        m[10] = -(zFar + zNear) / zRange; // POPRAWKA: dodany minus
        m[11] = -1.0f;
        m[14] = -(2.0f * zFar * zNear) / zRange; // POPRAWKA: dodany minus
        m[15] = 0.0f;
    }
}
```

### ➡️ Vector3f
```java
package utils;

// Nasza własna klasa reprezentująca punkt lub kierunek w przestrzeni 3D
public class Vector3f {
    public float x;
    public float y;
    public float z;

    // Konstruktor - ustawia wartości początkowe
    public Vector3f(float x, float y, float z) {
        this.x = x;
        this.y = y;
        this.z = z;
    }
}
```

## 🚀 Main.java
```java
import engine.GameLoop;

public class Main {
    public static void main(String[] args) {
        System.out.println("Uruchamianie CWL Simulator...");

        // Tworzymy instancję naszej pętli gry i ją startujemy
        GameLoop gameLoop = new GameLoop();
        gameLoop.start();
    }
}
```

## 📄 License
MIT
