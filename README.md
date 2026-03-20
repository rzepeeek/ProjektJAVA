# CWL SIMULATOR

## 📂 Struktura projektu
```text
src/
 |── engine/
 |   |── Camera
 |   |── GameLoop
 |   |── Input
 |   |── Mesh
 |   |── Renderer
 |   |── Shader
 |   |── Window
 |── entities/
 |   |── Car
 |   |── Ticket
 |── game/
 |   |── Game
 |   |── Player
 |── input/
 |── ui/
 |   |── HUD
 |── utils/
 |   |── Matrix4f
 |   |── Vector3f
 |── Main.java
```

## ⚙️ Engine
### Camera:
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

### GameLoop:
```java
package engine;

// import game.Game; // Zakładamy, że ta klasa powstanie wkrótce

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
        game = new game.Game();
        loop();
        cleanup();
    }

    private void loop() {
        // Zmienne do śledzenia czasu (Delta Time)
        long lastTime = System.nanoTime();
        double targetFPS = 60.0;
        double nsPerFrame = 1000000000.0 / targetFPS;
        double delta = 0;

        while (isRunning && !window.windowShouldClose()) {
            long now = System.nanoTime();
            delta += (now - lastTime) / nsPerFrame;
            lastTime = now;

            // Aktualizacja logiki gry (np. uciekający czas biletów, ruch kamery)
            while (delta >= 1) {
                update();
                delta--;
            }

            // Renderowanie (rysowanie na ekranie)
            render();

            // Aktualizacja okna (zamiana buforów)
            window.update();
        }
    }

    private void update() {
        if (Input.isKeyDown(org.lwjgl.glfw.GLFW.GLFW_KEY_ESCAPE)) {
            org.lwjgl.glfw.GLFW.glfwSetWindowShouldClose(window.getWindowHandle(), true);
        }

        game.update();

        // WYŚWIETLANIE STATUSU W TYTULE OKNA
        String status = "CWL Simulator | Punkty: " + game.getScore() +
                " | Mandaty: " + game.getFinesIssued() +
                " | " + game.getUiMessage();

        org.lwjgl.glfw.GLFW.glfwSetWindowTitle(window.getWindowHandle(), status);
    }

    private void render() {
        // Czyszczenie ekranu przed narysowaniem nowej klatki
        org.lwjgl.opengl.GL11.glClear(org.lwjgl.opengl.GL11.GL_COLOR_BUFFER_BIT | org.lwjgl.opengl.GL11.GL_DEPTH_BUFFER_BIT);

        if (game != null) {
            game.render();
        }
        // Tutaj w przyszłości będziemy rysować modele (Game.render())
        // np. game.render();
    }

    private void cleanup() {
        window.cleanup();
    }

}
```
